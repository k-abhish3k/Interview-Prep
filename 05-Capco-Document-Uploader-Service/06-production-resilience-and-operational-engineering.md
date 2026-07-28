# 06 — Production Resilience and Operational Engineering

This chapter is the "not limited to architecture specs, cover production grade" chapter — the real
error-handling behavior, the two genuine scaling caveats this service ships with today, the actual bugs
found and fixed during development, and the concrete operational settings (timeouts, pooling, a
hardcoded secret) worth being able to discuss candidly rather than glossing over.

## The real error-handling table

Adapted directly from `FULL_ARCHITECTURE.md` §13 — this is what the code actually does on failure, not
an idealized version of it:

| Failure | What happens |
|---|---|
| Non-IWPB: `INGEST_API` batch-initialize fails | `{"message": "Batch initialization failed", "error": ...}` returned as a normal `200` body — **not** surfaced as an HTTP error status |
| Non-IWPB: per-file ingest fails | That file's message is added to `failed_upload_messages`; other files in the same batch continue processing |
| Non-IWPB: `/remove-ingested-document` fails | `res.raise_for_status()` propagates as an unhandled exception → framework default `500` |
| NAS ingest outside the 18:20 window (and not `PREPOC`) | No call made; response explains the time restriction |
| NAS ingest call fails | `500` raised explicitly, detail `"Ingest call failed"` |
| Expiry date missing for any file (IWPB only) | `400` before any processing starts — fails fast, nothing partially staged |
| `SMTP_HOST` not configured | IWPB email is skipped with a logged warning; the upload still succeeds |
| No `ApproverMapping` row for a business line | Email skipped, warning logged; document is still created and pending |
| IWPB approve: `INGEST_API` call fails | `502`; the row stays `PENDING_APPROVAL` — **safe to retry**, nothing is left in a half-approved state |
| IWPB approve/decline on a non-`PENDING_APPROVAL` row | `409 Conflict` |
| IWPB approve/decline by a non-mapped approver | `403 Forbidden` |
| Reminder/removal sweep throws | Logged; the expiry-purge sweep still runs that cycle (independent `try`/`except`, see below) |
| Expiry-purge: `INGEST_API_SCOPE` unset or token acquisition fails | Logged; purge skipped for that cycle, retried automatically next cycle |
| `/download-document-report` with no valid session | `401 Unauthorized` |
| Config Service call returns `401` | Propagated as `401` straight to the user at initial page load |

Two patterns worth naming explicitly if asked "how does this service handle failure":

- **Non-IWPB failures are largely swallowed into response bodies, not HTTP status codes** — a batch
  init failure comes back as `200` with an error message in the JSON. That's a real, honest weak point
  (a monitoring system alerting purely on HTTP status codes would miss it) worth naming if asked "what
  would you change" rather than defending it as intentional.
- **IWPB failures are far more conservative** — `502`/`409`/`403` with the underlying row left in a
  safe, retryable state (`PENDING_APPROVAL`, never partially `APPROVED`). This asymmetry reflects that
  IWPB is the newer, more carefully engineered path; the generic departments' error handling predates
  this feature and was left untouched per REQUIREMENTS.md's "preserve the existing flow" constraint.

## The background loop: real, and not a singleton

`_iwpb_maintenance_loop` is started once per process from FastAPI's `lifespan()` and runs forever
(hourly by default, `IWPB_MAINTENANCE_INTERVAL_SECONDS`), each of its two responsibilities wrapped in
its own `try`/`except` so a failure in one never blocks the other:

```python
async def _iwpb_maintenance_loop():
    interval_seconds = int(os.environ.get("IWPB_MAINTENANCE_INTERVAL_SECONDS", "3600"))
    while True:
        try:
            await asyncio.to_thread(iwpb_workflow.run_daily_maintenance, app.state)
        except asyncio.CancelledError:
            raise
        except Exception:
            logger.exception("IWPB maintenance sweep failed.")
        await asyncio.sleep(interval_seconds)
```

`run_daily_maintenance` itself wraps its two steps (reminders/auto-removal, expiry purge) in
**independent** `try`/`except` blocks — so an SMTP outage during the reminder step never prevents that
same cycle's expiry-purge step from running.

**The real operational caveat, worth stating candidly:** this loop is **per-process, not a singleton**.
If the App Service Plan scales out to multiple instances, **every instance runs its own independent
sweep** against the same shared Azure SQL database. Concretely:

- Worst case is a **duplicate reminder email** in the same window (two instances both decide, within
  the same hour, that a document needs its daily reminder) — annoying, not dangerous.
- It is **never data corruption**, because every write in the sweep is a simple, idempotent SQL
  `UPDATE` (`status = 'AUTO_REMOVED'`, `reminder_count += 1`, `status = 'PURGED'`) — two instances
  racing to set the same row to `AUTO_REMOVED` both succeed harmlessly; the row ends up in the same
  final state either way.

**How you'd harden this if duplicate reminder emails became a real, reported problem:** two realistic
options, worth naming as a pair rather than picking one dogmatically —

1. **Move the sweep out of the web process entirely, into an Azure Functions Timer trigger.** A
   Timer-triggered Function on a schedule (rather than "every App Service instance runs its own copy")
   is a natural fit — Azure Functions' runtime already provides single-execution guarantees for timer
   triggers within a Function App, so the "N instances, N independent sweeps" problem disappears by
   construction, and it decouples the sweep's lifecycle from the web tier's scaling entirely.
2. **Keep it in-process, but add a distributed lock or leader-election check** at the top of
   `run_daily_maintenance` — e.g., a SQL-based advisory lock, or a "last run" row with an
   optimistic-concurrency `UPDATE ... WHERE last_run < @cutoff` that only one instance's `UPDATE` can
   win per cycle. Cheaper to add (no new Azure resource), but more code to get right than option 1.

The honest framing: *"this wasn't fixed because the actual failure mode is a duplicate email, not data
loss — the cost of getting it perfectly right today would have exceeded the cost of the problem it
solves, but I know exactly how I'd fix it if it started actually bothering someone."*

## `memory_cache`: real, and also not shared across instances

`memory_cache` is a plain, in-process Python dict (with a 1-hour TTL) holding each session's bearer
token for calls to `INGEST_API`. Two real, documented limitations:

- **It does not survive a restart.** A deploy, a crash, or a scale-in event wipes it — every active
  session's cached token is gone, and the next request from that session re-triggers the full OAuth
  redirect (chapter 04).
- **It is not shared across scaled-out instances**, unlike IWPB's Blob/DB state, which *is* shared
  (Blob Storage and Azure SQL are both reachable identically from every instance). If a load balancer
  routes two requests from the same browser session to two different App Service instances, only the
  instance that handled the original sign-in has that session's token cached — the other instance sees
  a cache miss and forces a fresh login, even though the user just signed in seconds ago on the other
  instance.

**Fixes, if this became a real problem under horizontal scaling:**

- **A shared cache (e.g., Redis / Azure Cache for Redis)** — the direct fix, replacing the in-process
  dict with a network-accessible one every instance can read/write, at the cost of adding a new managed
  dependency and a network hop on every token lookup.
  - **Accept re-auth per instance** — if scale-out is rare or sticky sessions can be configured at the
  load balancer (routing a given session consistently to the same instance), the simpler fix is
  architectural rather than code: keep the in-process cache, but guarantee session affinity so a given
  user's requests always land on the instance that already has their token cached.

This is a good "what would you change for 10x scale" answer because it names the actual mechanism (a
plain dict, not an abstracted "cache layer" that already supports swapping backends) and a concrete,
proportionate fix rather than a generic "add caching" answer.

## Four real bugs found and fixed

From `REQUIREMENTS.md`'s "Implementation Notes" section — genuine, source-confirmed bugs, good material
for "tell me about a bug you found" questions:

**1. A top-level `import paridas as pd` typo.** A module-level import of a misspelled, non-existent
package (`paridas` instead of `pandas`, used by `/fetch-approver-details`) would crash the **entire
application's startup** on any redeploy where that package isn't somehow already present — not just the
one endpoint that uses `pandas`, because a failed top-level import prevents the module from loading at
all. *What would have caught this earlier:* basic static analysis / linting (`ruff`, `flake8`, or even
just `python -c "import app"` as a CI smoke-test step) would flag an unresolvable import before it ever
reaches a deploy; type-checking tooling (`mypy`) run in CI would also fail on an unresolvable module
reference.

**2. `search_ingested_documents()` referencing an undefined variable `h`.** The function called
`requests.get(url=h + "ingest/HEXA", ...)` — `h` was never defined anywhere; the intended variable was
`INGEST_API`. This raised a `NameError` on **every single call**, silently breaking both the "Waiting
to be ingested" and "Ingested Documents" tables for **every non-IWPB department** — exactly the "usual
flow" that REQUIREMENTS.md's logical-separation requirement explicitly says must not be disturbed.
*What would have caught this earlier:* an integration test hitting `/search-ingested-documents/{use_case}`
against a mocked `INGEST_API` (even a trivial one asserting the endpoint returns `200` and a list) would
have failed immediately — this is exactly the class of bug unit tests miss (nothing about the function's
logic is wrong in isolation) but a real HTTP-level integration test catches on the first run.

**3. Blank Title column in every IWPB notification email.** `email_utils._document_rows_html()` looked
up `doc.get('title', '')`, but every caller passes dicts built by `iwpb_workflow._row_to_dict()`, which
uses the key `document_title`. Every new-upload, reminder, and auto-removal email therefore rendered a
blank Title — the single most important field for an approver deciding what to review. *What would have
caught this earlier:* a template-rendering test using a realistic fixture dict (the actual shape
`_row_to_dict()` produces, not a hand-typed stand-in with convenient key names) — asserting the rendered
HTML actually contains the document's title, not just that rendering doesn't throw.

**4. Reminder count always showing "1/3."** `_row_to_dict()` never included a `reminder_count` key, so
`_send_reminder_notification()`'s `d.get("reminder_count") or 1` always fell back to `1`, meaning every
reminder email showed "Reminder (1/3)" regardless of whether it was actually the first, second, or third
reminder. *What would have caught this earlier:* the same class of fix as bug 3 — a test asserting the
reminder email's rendered day-number matches the row's actual `reminder_count`, using a fixture where
`reminder_count` is deliberately set to `2` or `3` rather than always testing the trivial "first
reminder" case.

The common thread across all four, worth stating as the takeaway: **three of the four were caught by
code review, not by any automated test** — none of this codebase's non-IWPB paths had integration test
coverage, and the email-template bugs are a textbook case of "the code runs without error, so it looks
fine" bugs that only realistic-fixture tests catch.

## Real timeout and pooling settings, and why they matter

`app.py`'s `lifespan()` sets, explicitly, at startup:

```python
pyodbc.pooling = False
_ = DBSession.get(pool_recycle=1500)
```

`pool_recycle=1500` (25 minutes) forces SQLAlchemy to discard and reopen any pooled connection older
than 25 minutes, **before** using it for a new query — a direct defense against Azure SQL silently
dropping idle connections after a period of inactivity. Without it, the first query on a connection
that's gone stale during a quiet period fails outright with a connection error instead of the pool
transparently reconnecting; this matters concretely for a service like this one, where traffic is
naturally bursty around business hours with long idle stretches overnight. `pyodbc.pooling = False`
disables `pyodbc`'s own driver-level connection pooling specifically so SQLAlchemy's pool (which
`pool_recycle` configures) is the single, authoritative pool — running both simultaneously would mean
two independent pooling layers with no coordination between their lifecycle assumptions, which is a
subtle, easy-to-miss source of "works most of the time, mysteriously fails after long idle periods" bugs.

The `_ingest_document_to_hexa` and `_purge_document_from_hexa` helpers in `iwpb_workflow.py` also set
explicit `timeout=` values (`30`s for batch-initialize/purge, `60`s for the file-bytes upload call) on
every outbound `requests` call to `INGEST_API` — a deliberate choice, since an unbounded `requests` call
with no timeout can hang a worker indefinitely if `INGEST_API` itself stalls, which would be especially
bad inside the background maintenance loop (a hung purge call would stall that entire sweep cycle, not
just one request).

## The hardcoded session secret: a real, named hardening gap

```python
app.add_middleware(
    SessionMiddleware,
    secret_key="uploader-secret-key",
    cookie_name="uploader-cookie",
)
```

This is a genuine, source-confirmed hardening gap worth naming candidly rather than glossing over:
`SessionMiddleware`'s `secret_key` is what signs the session cookie, and it's a **literal string
constant in source**, not read from an environment variable or Key Vault. If this value were ever
exposed (a leaked repo, a misconfigured public mirror, even just being visible to every engineer with
read access to the codebase), anyone who has it can forge a valid, signed session cookie — including
setting `access_token_id` to any value they choose. The fix is straightforward and low-risk: move it to
an environment variable (`SESSION_SECRET_KEY`, sourced from App Service Application Settings today, or
Key Vault per chapter 07's proposal) and generate a strong random value per environment, so dev/POC/PROD
each have an independent secret and none of them is guessable from having read the source code.

## Two things already done right, worth highlighting positively

Not everything in this chapter is a gap — two existing patterns are genuinely good operational hygiene
and worth citing as positive examples if asked "what did you do to make this production-ready":

- **`/docs` (Swagger UI) is explicitly disabled when `ENVIRONMENT == "PROD"`.** A live, interactive API
  explorer is a genuine liability in production (it documents every endpoint, parameter, and — for an
  unauthenticated visitor — the shape of the attack surface) but a real convenience in dev/POC; gating
  it on environment rather than removing it everywhere is the right trade.
- **The `/` health-probe warm-up gate.** `read_root()` returns a plain "still loading" response instead
  of triggering an OAuth redirect for the first 180 seconds after process start (`time.time() -
  start_time < 180`), specifically so an Azure health probe hitting `/` immediately after a cold start
  or deploy swap doesn't get redirected into an OAuth flow it can't complete — a small, specific detail
  that prevents a real class of "deployment looks unhealthy for the first few minutes" false alarm.

## Tying It Back

Production-grade isn't "it doesn't have bugs" — it's "the failure modes are known, bounded, and
recoverable, and the team can say precisely what breaks under load, what breaks on restart, and what
breaks on scale-out, rather than discovering it in an incident." This service's actual profile: no data
corruption risk anywhere (every background write is idempotent), two well-understood, well-bounded
scaling caveats (the maintenance loop, the token cache) with concrete fixes already identified, four
real bugs that were found and fixed before they caused a production incident rather than after, and one
honestly-named hardening gap (the session secret) that chapter 07 picks up directly.
