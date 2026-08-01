# 99 — Interview Q&A: Document Uploader Service

Read this last, once chapters 01–08 and the notebooks are fresh. Questions are grouped by type; in a
real interview they'll be mixed together, often as follow-ups to each other. **Q1 is the actual
question the candidate was asked in a real interview about this project** — it comes first,
deliberately, ahead of the general behavioral warm-up questions.

---

## Q1. For revised versions of the same document, how are you handling those?

This is the real question. The full worked answer lives in chapter 05
(`05-document-lifecycle-versioning-and-revisions.md`) — this is the compressed, interview-ready
version:

> "Today, there's no versioning in this system — every upload, in every department, creates a
> brand-new document with a brand-new ID; there's no title-based dedup or supersede logic anywhere in
> the upload path. I confirmed that directly by reading the upload handler: every call goes through
> `INGEST_API`'s batch-initialize-then-ingest flow and gets back a fresh document ID every time, with
> no lookback for an existing document with the same title.
>
> What you *can* do today, with existing endpoints, is a manual three-step workaround: search by title
> and business line to find the stale version, remove it via the existing delete endpoint, then upload
> the revision. The pieces all exist and already work — the gap is that it's entirely client-driven,
> not automated or enforced, and you lose the original document ID in the process, so anything that
> referenced it breaks.
>
> It's also worth being precise that this is a different concern from the approval workflow one of our
> departments (IWPB) has — that state machine tracks whether one specific upload has been approved and
> is within its retention window, not whether it's the current revision of a document that's had
> several versions. Those are two different axes, and I'd be careful not to conflate them.
>
> If I were building real revision tracking, I'd add a `document_group_id` and a
> `supersedes_document_id` to the document model, so 'get the latest version' and 'show me the
> revision history' become well-defined queries instead of ambiguous title searches. But that only
> solves half the problem — the ingestion API this service calls doesn't have a version concept
> either today, so a complete fix needs that system to understand versioning too. And if there's a
> search index behind retrieval downstream of this, which is likely, a real revision feature also
> needs to retire the old version's index entry when a new one supersedes it. Otherwise you've solved
> the bookkeeping problem, but a RAG system on top could still serve stale content."

**Follow-ups to be ready for:**

- *"So nothing stops two people uploading conflicting revisions right now?"* Correct, and worth saying
  plainly: there's no locking, no "someone else already revised this" check, nothing. The manual
  workaround is entirely coordination-by-convention.
- *"Would you say that's a gap in the system?"* Yes, honestly — it's a gap that was never in scope for
  the work that was actually delivered (the IWPB approver workflow), not an oversight within that
  scope. Naming it as "out of scope for what was asked" rather than "a bug I missed" is accurate and
  more credible than either overclaiming it's handled or apologizing for it.
- *"How would you prioritize building that vs. other things?"* A reasonable answer: it depends on how
  often revisions actually happen, and how costly a stale duplicate is downstream (a stale search
  result surfaced in a RAG answer is a real, painful failure mode) — worth quantifying with real usage
  data before committing to the two-layer fix (this service + `INGEST_API`) described above.

---

## Behavioral

### Q2. Tell me about the Document Uploader Service you built at Capco.

Use the STAR summary from `00-README.md`, under 90 seconds:

- **Situation** — HSBC needed document ingestion across several business lines, with IWPB specifically
  needing human approval, an audit trail, and expiry-driven retention that no other department needed.
- **Task** — extend an existing FastAPI service with an IWPB-only approver workflow, without disturbing
  the other departments' working flow.
- **Action** — built `iwpb_workflow.py` as a separate module with its own SQL table, staged uploads in
  Blob Storage (not local disk, so it survives restarts and works across scaled-out instances), SMTP
  notification to mapped approvers, a background `asyncio` sweep for reminders/auto-removal/purge with
  independent error isolation per responsibility, and a separate, relationship-based authorization
  pattern for approver identification — while finding and fixing several real bugs along the way (a
  startup-crashing import typo, a silent `NameError` breaking search for every other department, two
  email-template key mismatches).
- **Result** — a real, auditable approval and retention lifecycle for IWPB, with zero disruption to the
  other five departments' existing flow.

### Q3. Describe a technical decision on this project you'd defend, and one you'd reconsider.

**Defend:** staging IWPB uploads in **Azure Blob Storage rather than local/App Service disk**.
`REQUIREMENTS.md`'s Implementation Notes confirm local-disk staging was the *initial* implementation,
deliberately replaced with Blob Storage specifically so staged files survive a restart and are visible
from every scaled-out App Service instance, without depending on the `/home` file share. That's a real
example of catching a horizontal-scaling problem before it shipped, not after.

**Reconsider:** the `memory_cache` bearer-token cache and the `_iwpb_maintenance_loop` background task
are both per-process, not shared/coordinated across scaled-out instances (chapter 06) — unlike the
Blob/DB decision above, these weren't revisited before shipping. Worth naming candidly as the kind of
thing you'd fix given more time, with the specific fixes already identified (a shared cache, an Azure
Functions Timer trigger, or a distributed lock) rather than left vague.

### Q4. How did you handle a real bug you found in this service?

Pick one of the four from chapter 06/`REQUIREMENTS.md` and walk it with STAR — the `NameError` one is
the strongest, most concrete example:

- **What broke** — `search_ingested_documents()` referenced an undefined variable `h` instead of
  `INGEST_API`, raising a `NameError` on every call.
- **Impact** — both the "Waiting to be ingested" and "Ingested Documents" tables were silently broken
  for every non-IWPB department.
- **How found** — code review while implementing the IWPB feature, reading the existing function it
  had to coexist with.
- **Fix** — a one-line correction to `INGEST_API`.
- **What would catch it earlier** — a basic integration test against a mocked `INGEST_API`, asserting
  the search endpoint returns `200`, since this is exactly the class of bug that only manifests at
  runtime, not in isolated unit logic.

---

## Real Departments, RBAC, and Feature Flags

### Q5. What are the actual business lines/departments this service supports?

**IWPB, FEMA, TPMB, and GTRM**, plus a generic bucket (RESEARCH, WCS, GPS, WCL, OTHER, GENERAL) with no
dedicated role. **There is no "Credit Ops"** — if that name surfaces from memory, correct it in the
moment. IWPB is the only one with an approval workflow; the other three are direct, unmediated proxies
to `INGEST_API`, gated by AAD roles (`stitt.ingester`, `.pilot`, `.tpmb`, `.gtrm`) and feature flags
(chapter 03 has the full table).

### Q6. Walk me through the `/role-check` logic — how does a user's role become a list of visible business lines?

1. Map each AAD role the user holds to its business line (`stitt.ingester → IWPB`, `.pilot → FEMA`,
   `.tpmb → TPMB`, `.gtrm → GTRM`).
2. If `FEATURE_THIRD_PARTY_BOT` is on **and** the user holds all three of
   `stitt.ingester`/`.pilot`/`.tpmb`, return `[IWPB, FEMA, TPMB]` directly.
3. If `FEATURE_GREEN_TIME` is on **and** the user holds all four roles, return
   `[IWPB, FEMA, TPMB, GTRM]`.
4. Otherwise, return the sorted list from the direct one-role-per-line mapping.

The key nuance: TPMB and GTRM aren't just "one more role" — they require holding every role beneath
them in the hierarchy *and* their own feature flag being on, which reads as a deliberate
progressive-rollout pattern layered on top of AAD role assignment. `notebooks/04_role_to_business_line_rbac_demo.ipynb` implements this exact algorithm and exercises it against several
role-set/flag combinations.

### Q7. Why one shared service instead of a microservice per department?

The departments differ enormously in how much state they actually need: IWPB genuinely warrants its
own module (which is what it got — `iwpb_workflow.py`, its own SQL table, its own background job),
while FEMA/TPMB/GTRM are thin, near-identical proxies sharing one `if` branch's worth of difference
from the generic departments.

Splitting into four services would have meant four deployment pipelines, four sets of Application
Settings, and either duplicating or awkwardly sharing the `INGEST_API` client and `/role-check` logic
across service boundaries — for departments that don't hold any state of their own to protect from each
other. The one place isolation genuinely matters (IWPB's extra complexity) already has it, at the
module level, without needing a service boundary: `iwpb_workflow.py` only ever touches
`IWPBDocumentWorkflow`, so a bug there can't corrupt a FEMA/TPMB/GTRM upload even inside the same
process. Chapter 03 has the full argument, including the case *for* splitting them, argued fairly.

### Q8. Is the department/business-line RBAC in this service the same thing as the "4 Azure AI Search indexes per department" retrieval setup you've mentioned elsewhere?

No, and it's important not to conflate them. This service's RBAC (`/role-check`) fully controls which
departments a user can upload to and see in this app — that's confirmed, real, and covered in chapter
03/04. Whatever HEXA or a downstream retrieval layer does with per-department search indexes and access
control is **not visible in this repository** — this service's job stops at handing a file to
`INGEST_API`. The honest, defensible connection: the department entitlement data this service already
computes is exactly the kind of signal a downstream per-department search-ACL model would need to
consume — but that's a statement about what would naturally connect, not a claim about code that
exists in this codebase.

---

## Authentication, RBAC, and the Manual-OAuth-vs-MSAL Nuance

### Q9. Does this service use MSAL for OAuth 2.0 authentication?

Not for the part most people assume. User sign-in is a **hand-rolled authorization-code exchange**:
`app.py` builds the Azure AD `/authorize` redirect URL by hand, and exchanges the returned `code` for a
token via a raw `requests.post` to the `/token` endpoint — not via `msal.PublicClientApplication` or
any MSAL authorization-code helper.

MSAL (`msal.ConfidentialClientApplication`) *is* used, but for two narrower things: **on-behalf-of**
token acquisition (`acquire_token_on_behalf_of`, to call the feature-flag Config Service using the
signed-in user's own token) and **client-credentials** token acquisition (`acquire_token_for_client`,
for the IWPB background purge job, which runs with no user in the loop).

Getting this right — naming the split instead of saying "MSAL handles OAuth" — is a stronger, more
precise answer. Chapter 04 has the full breakdown with the actual code for each of the three
mechanisms.

### Q10. How does this service know a caller's roles — does it validate a JWT the standard way?

Not quite — this is worth being precise about. `utils.get_user_role` decodes the
`x-ms-token-aad-id-token` header with `jwt.decode(..., options={"verify_signature": False})` —
explicitly unverified. There's no signature check, `aud` check, `iss` check, or `exp` check on this
token inside this app's own code.

The design relies on a trust boundary upstream of this FastAPI process — the header is the kind Azure
App Service's built-in "Easy Auth" or a gateway would inject after already validating the identity. So
the honest answer is: this app trusts that whatever sits in front of it has already verified the token
and guarantees the header can't be spoofed by an external caller; it does not independently re-verify
that itself.

### Q11. IWPB has an "approver" concept — is that the same as an AAD role?

No — and this is a good one to volunteer as a nuance. Business-line **visibility** (can a user even see
IWPB in their dropdown) is role-based RBAC, driven by the `stitt.ingester*` AAD app roles. Whether a
specific user can **approve or decline an IWPB document for a given business line** is a completely
separate, **relationship-based** pattern: their email (from a different header,
`x-ms-client-principal-name`) has to match `approver1_email_id`/`approver2_email_id` in an `ACTIVE`
row of the `approver_mapping` SQL table for that business line. No AAD role governs approver status at
all — it's entirely data-driven, self-service through `/save-approver-details`, with no distinct admin
role even gating who can set it. Chapter 04 has the full comparison table.

### Q12. How would you test the RBAC/role-check logic without hitting real Azure AD in CI?

`role_to_business_line` and the feature-flag gating logic are pure functions of `(user_roles,
feature_flags)` once the JWT is decoded — they don't need a real token at all to unit test, just a
Python list of role strings and a couple of booleans, exactly as demonstrated in
`notebooks/04_role_to_business_line_rbac_demo.ipynb`.

The one part that *does* need care in testing is `get_user_role`'s unverified `jwt.decode` call —
since it never validates a signature, a test can hand it a locally-constructed, entirely unsigned token
and it will decode successfully. That's itself worth demonstrating in a test, as a way of documenting
the trust-boundary assumption explicitly, rather than leaving it implicit.

---

## Production Resilience and Scaling

### Q13. This service can scale to multiple App Service instances — what breaks, and what doesn't?

Two things behave differently under scale-out, and it's worth naming both without conflating them.

- **Blob Storage and Azure SQL are shared and safe.** Any instance can serve an approve/decline for a
  document staged by any other instance, because neither depends on local process/disk state.
- **Two things are per-process and *not* shared:** the `memory_cache` bearer-token cache (a session's
  cached token only exists on the instance that handled sign-in — a request routed to a different
  instance forces re-login) and `_iwpb_maintenance_loop` (every instance runs its own independent
  hourly sweep against the same database).

The maintenance-loop risk is bounded — worst case is a duplicate reminder email, never data corruption,
since every write in the sweep is an idempotent SQL `UPDATE`. Fixes for both are concrete: a shared
cache (Redis) or session affinity for the token cache; an Azure Functions Timer trigger or a
distributed lock for the maintenance loop. Chapter 06 has the full detail.

### Q14. Tell me about two bugs you found and fixed in this service.

Pick two from chapter 06's four. Strongest pair:

1. **The top-level `import paridas as pd` typo** — a misspelled, non-existent package import that would
   crash the *entire app's startup* on redeploy, not just the one endpoint that used it; caught by basic
   static analysis/linting or a CI import smoke-test.
2. **The blank Title column in every IWPB approver email** — `email_utils._document_rows_html()` looked
   up `doc.get('title', '')`, but every caller's dict used the key `document_title`, so the single most
   important field for an approver deciding what to review was blank on every notification, reminder,
   and auto-removal email; caught by a template-rendering test using a realistic fixture (the actual
   dict shape `_row_to_dict()` produces), not a hand-typed convenient one.

The pattern worth naming across both: neither was caught by an automated test, because the non-IWPB
paths and the email templates had no integration or rendering test coverage at all — a concrete,
specific answer to "what would you add" rather than a generic "more tests."

### Q15. How would you monitor and alert on this service in production?

Beyond generic latency/error-rate monitoring:

- Alert on documents stuck in `PENDING_APPROVAL` past a reasonable window — a signal the SMTP
  notification or the reminder sweep itself is failing silently (both are best-effort, and swallow
  their own exceptions, chapter 06).
- Alert on the expiry-purge step being skipped repeatedly (logged clearly when `INGEST_API_SCOPE` is
  missing or token acquisition fails — a string worth alerting on directly).
- Specifically watch for **duplicate reminder emails to the same recipient within the same window**, as
  the concrete, observable symptom of the background-loop not-a-singleton caveat actually manifesting
  under real scale-out, rather than staying theoretical.

---

## Secrets, Config, and Deployment

### Q16. Is this service's configuration in Key Vault?

No — confirmed from the real deployment docs: it's in **Azure App Service Application Settings**
(environment variables), with `.env` explicitly local-dev-only. Managed Identity *is* real, but it's
used for Blob Storage access (`DefaultAzureCredential` against the storage account), not for
retrieving secrets from a vault.

If asked to propose an improvement: migrate the genuine plaintext secrets —
`MICROSOFT_PROVIDER_AUTHENTICATION_SECRET`, `SMTP_PASSWORD`, `AZURE_STORAGE_CONNECTION_STRING` if used,
and especially the **hardcoded `SessionMiddleware` secret_key** currently sitting as a literal string in
source — to Key Vault references resolved through the same Managed Identity, which requires zero code
changes, since `os.environ[...]` reads would resolve the injected value transparently either way.

### Q17. Does this service run in Docker?

Not in production — this is a real correction worth making plainly if it comes up, since it's a
natural assumption for a FastAPI service. It runs on **Azure App Service's native Python runtime**
(`uvicorn`/`gunicorn` running `app.py` directly), with no Dockerfile, no container registry, and no Web
App for Containers involved. Containerizing it is a reasonable, low-risk proposal to make in an
interview — environment consistency, a path to other compute targets — but it should be framed
explicitly as a proposal, not a description of what's running today.

### Q18. What's the one hardcoded secret in this codebase you'd fix first?

`SessionMiddleware(secret_key="uploader-secret-key", ...)` in `app.py` — a literal string constant that
signs the session cookie. Anyone with that value can forge a valid signed session cookie, including
setting arbitrary session state. The fix is low-risk and mechanical: move it to an environment variable
(sourced from Application Settings today, Key Vault as the proposed hardening step), generate an
independent strong random value per environment, and nothing else in the app needs to change.

---

## "What Would You Change If You Rebuilt This Today?"

### Q19. What would you change if you rebuilt this today?

Three specific, defensible changes, each already argued in depth elsewhere in this course:

1. **Real document revision tracking** (chapter 05) — a `document_group_id`/`supersedes_document_id`
   pair, plus the harder, cross-team half: getting `INGEST_API`/HEXA itself to understand versioning,
   since this service alone can't fully solve it.
2. **Move the background maintenance sweep out of the web process** — an Azure Functions Timer trigger
   (or a distributed lock if staying in-process), to eliminate the per-instance-not-singleton caveat
   before it causes a real, reported problem rather than after.
3. **Close the two named hardening gaps together** — the hardcoded session secret and the plaintext
   `MICROSOFT_PROVIDER_AUTHENTICATION_SECRET`/`SMTP_PASSWORD` values, via the Key Vault-reference
   migration in chapter 07 — a small, low-risk, high-value change that's been identified but not done.

The underlying point: naming real, specific technical debt with a concrete fix already in mind is a
stronger signal than either "nothing, it was perfect" or a vague "I'd use better technology."
