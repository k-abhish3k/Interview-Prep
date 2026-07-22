# 07 — Production Resilience and Operational Engineering

This chapter does for the monitoring platform what Course 03's chapter 06 does for the document
uploader: not architecture-diagram-level description, but the real, concrete behavior under failure —
which evaluator failures are allowed to reach a user-facing outcome versus which are quietly logged, the
genuine concurrency caveat this kind of pipeline ships with, illustrative "bug found and fixed"
narratives specific to an evaluation platform, and the concrete operational numbers (timeouts, retries,
thresholds) worth being able to state and defend rather than gesture at vaguely.

As with chapter 06, everything below is an **illustrative, plausible reconstruction** — there is no real
source repository backing this course, unlike Course 03. Treat the specific numbers, table rows, and bug
narratives as "here's the level of detail I'd defend if I'd built this," not as verified fact recalled
from a real codebase.

## The error-handling table: which evaluator failures block a response, and which just get logged

The organizing principle, established in chapter 01 and reused here: **Tier 1 (synchronous, cheap,
gates the response) fails differently than Tier 3 (asynchronous, sampled, feeds the dashboard).** A
failure in the wrong tier, handled the wrong way, is a real production risk either direction — fail
closed on an async evaluator and you've turned a monitoring outage into a user-facing outage; fail open
on a synchronous safety gate and you've turned an evaluator bug into a live moderation gap.

| Evaluator / stage | Failure mode | What happens | Blocks the response reaching the user? |
|---|---|---|---|
| Tier 1 PII/toxicity regex gate (synchronous, 100% of traffic) | Raises an exception (malformed input encoding, unexpected null field) | **Fails closed** — the response is held, a generic safe fallback ("I'm not able to help with that right now") is returned instead, and the exception is logged with full context for review | Yes |
| Tier 1 PII/toxicity regex gate | Exceeds its latency budget (50ms) without erroring | **Fails open** — passes the response through unflagged rather than blocking on a rare slow-path, but logs a `tier1_timeout` event distinctly from a normal pass, so a spike in timeouts is itself visible on the dashboard even though no single timeout blocked anything | No — but tracked as a gap, not silently dropped |
| Rule-based harmfulness classifier (gradient-boosted, chapter 03) | Model fails to load, or raises on inference | **Depends entirely on which tier it's wired into** — if it's part of the Tier 1 synchronous path, fails closed like the regex gate above; if it's running in the async Tier 3 sample, fails open (logged only). This ambiguity is itself worth naming explicitly as a real design risk: the same classifier code path behaving differently depending on which tier calls it is exactly the kind of thing that causes an incident when someone refactors the pipeline and forgets which mode a given call site expects | Depends on call site — see bug narrative 2 below |
| LLM-as-judge (completeness, hallucination, chapter 01) | API call raises — timeout, rate limit (`429`), malformed judge output that doesn't parse as the expected rubric shape | **Never blocks, by design** — this evaluator only ever runs asynchronously on a sample, after the response has already reached the user (chapter 01). Failure logs an `evaluation_failed` record against that `response_id` and the sample is excluded from that metric's aggregate for the period, distinctly from a genuine low score | No — cannot block; the response left the building before this evaluator ran |
| Ragas faithfulness / answer relevancy scorer | Malformed input — empty retrieved-context list (a genuine retrieval miss, not a formatting bug) | See bug narrative 1 below — this was a real, source-of-truth-distorting bug, not a hypothetical | No — but silently corrupted the metric it was supposed to protect |
| Per-client hourly aggregation job (rolls per-response records into dashboard trend tables) | Fails mid-run (a transient Azure SQL throttling error, a partial write) | Logged; the job retries on its next scheduled run. Because the job writes to a staging table and only atomically swaps it into the dashboard-facing table on full success — never updates the dashboard table incrementally in place — a failed run leaves the dashboard showing the last *fully successful* rollup rather than a partial or corrupted one | No — user-facing assistant unaffected; only dashboard freshness (never correctness) is at risk |
| Bad-actor/adversarial regression suite (chapter 04, CI gate on every deploy) | A test in the suite **errors** (a malformed fixture, an unhandled exception in the harness itself) rather than cleanly **failing** (assertion mismatch) | The pipeline treats an erroring test identically to a failing one — it blocks the deploy. This is a deliberate choice: silently treating "the test itself broke" as "the test passed" would be exactly the kind of gap a bad-actor regression could quietly slip through | Blocks a *deploy*, not a live response |
| Feedback ingestion (thumbs up/down, reliance signals) | Malformed or missing `session_id` on an inbound feedback event | Logged and dropped from aggregation; feedback is treated as a best-effort signal (chapter 04 already frames it as inherently sparse/noisy), so a dropped malformed event degrades signal quality slightly rather than being treated as an incident | No |
| Threshold/alert evaluation itself (the job that checks rolling baselines against the 2-std-dev rule, chapter 05/06) | Throws (e.g., a `NULL` baseline row for a brand-new `assistant_version_tag` still in `LEARNING` state, chapter 06) | Logged; that specific client/metric's alert check is skipped for the cycle rather than the whole alerting job crashing — each client/metric pair's evaluation is wrapped independently, mirroring the "one failure shouldn't take down unrelated work" pattern | No — and specifically doesn't take down alerting for *other* clients or metrics |

Two patterns worth naming explicitly if asked "how does this pipeline handle failure":

- **The user-facing path (Tier 1) is deliberately conservative and fails closed on error, open on
  timeout** — an evaluator bug should never silently let something unsafe through, but an evaluator
  being briefly slow shouldn't turn into an availability incident for the whole assistant either. That
  asymmetry (fail closed on exception, fail open on timeout, both logged distinctly) is the honest
  design, not a single blanket rule.
- **Everything past Tier 1 fails open by construction, because it runs after the user already has their
  answer** — the only lever left once you're in Tier 3 is "how fast do we notice and how loudly do we
  say so," not "can we stop this response," which is exactly why the metrics-store and alerting
  correctness (bug narratives 1, 3, and 4 below) matter as much as they do: a silent corruption in an
  async evaluator doesn't just miss one bad response, it can quietly distort the trend the whole risk
  team is relying on.

## The concurrency caveat: two evaluators racing on the same response, and a job that isn't a true singleton

**Two independent evaluators can write to the same per-response metric record concurrently, and a naive
implementation corrupts the count.** The rule-based PII/toxicity gate (Tier 1, synchronous) and the
LLM-as-judge harmfulness classifier (Tier 3, asynchronous, running some seconds later on a worker
process) can both independently decide to flag the *same* `response_id`. If each evaluator's flagging
logic is implemented as an application-level read-modify-write —

```python
# The naive, race-prone version
current = get_flagged_count(client_id, day)
set_flagged_count(client_id, day, current + 1)
```

— then two evaluators flagging the same response within the same window can genuinely double-count it:
evaluator A reads `current = 41`, evaluator B reads `current = 41` before A's write lands, both write
`42`, and the true count of 43 (41 plus two independent flags) is lost to 42. This isn't a rare,
theoretical race — it's the direct consequence of Tier 1 and Tier 3 deliberately running on separate
execution paths (one synchronous in the request path, one asynchronous on a worker pool) precisely
*because* chapter 01 argues they should never block each other. The same design decision that makes the
pipeline fast enough to run in production is what creates this race. See bug narrative 4 below for what
this looks like when it actually manifests, and the fix (atomic SQL increment, or better, counting
distinct flagged `response_id`s instead of incrementing a counter at all).

**The per-client hourly aggregation job is not a true singleton across scaled compute.** If the
aggregation step (chapter 05's "aggregation jobs (hourly/daily) -> dashboard tables") is deployed as a
scheduled job on more than one compute instance — plausible if it runs as a container job or an Azure ML
pipeline step with more than one node, rather than a single dedicated scheduler — every instance can
independently decide "it's the top of the hour, time to roll up the last hour's records" and run the
same rollup concurrently. Because the job writes to a staging table before atomically swapping it in (see
the error-handling table above), the worst case is *not* a corrupted final table — it's wasted compute
(the same hour rolled up twice, redundantly) and, if the swap-in step itself isn't idempotent, a narrow
window where two swaps race and whichever completes last silently wins, discarding the other's otherwise
identical work. This is a lower-severity version of Course 03's not-a-singleton background-loop caveat
(chapter 06 there): the fix is the same shape — either move the job to a scheduling mechanism that
natively guarantees single execution (an Azure Functions Timer trigger, or a single dedicated scheduler
instance rather than a scaled-out compute pool), or add a lightweight distributed lock (a
`SELECT ... FOR UPDATE`-style row lock, or an optimistic `UPDATE ... WHERE last_run_at < @cutoff` that
only one concurrent caller's write can win) at the top of the rollup function.

## Four illustrative bugs found and fixed

Presented in the same spirit as Course 03's four real bugs — concrete, specific, and structured around
"what would have caught this earlier" — but explicitly labeled as **illustrative** scenarios
representative of the kind of bug this class of system produces, since there is no source repository to
confirm these against.

**1. A threshold misconfiguration that flooded an on-call channel with false positives overnight.** The
harmfulness-flag-rate alert was originally written as a flat "alert if flag rate > 20% in the trailing
window" with no minimum sample-count gate. During a low-traffic overnight window, 2 of 6 sampled
responses tripped the harmfulness classifier — both borderline scores just over a slightly aggressive
threshold, not a real safety incident — and 2/6 = 33% breached the 20% rule, paging the on-call rotation
at 3 a.m. for what was, in absolute terms, two borderline responses. *Fix:* require a minimum sample
count (proposed: n >= 50 in the window) before a percentage-based rule is allowed to fire at all; below
that floor, fall back to a raw-count threshold instead ("page only if 5+ responses are flagged," not a
percentage of a handful of samples) — exactly the volume-aware thresholding chapter 05 argues for in
principle, applied here as the concrete fix for a concrete incident. *What would have caught this
earlier:* backtesting the proposed alert rule against several weeks of historical traffic — including
the known overnight low-volume period — before turning it on live, rather than validating it only
against a representative business-hours sample.

**2. A Ragas-style faithfulness scorer silently returning `None` on malformed input, and that `None`
being treated as a passing score.** `compute_faithfulness(answer, context)` returned `None` when
`context` was an empty list — a legitimate outcome of a retrieval miss (the retriever genuinely found
nothing for that query), not a bug in the scorer itself. The aggregation code, however, was written as
`if faithfulness_score and faithfulness_score < THRESHOLD: flag_response(...)` — and because `None` is
falsy in Python, `None and (None < THRESHOLD)` short-circuits to `None` without ever raising or flagging.
The practical effect: every zero-retrieval response silently skipped faithfulness evaluation *and* was
excluded from both the numerator and denominator of the rolling faithfulness average — quietly inflating
the metric's apparent health by removing exactly the population of responses (retrieval misses) most
likely to be genuinely unfaithful, from the one metric meant to catch that. *Fix:* treat `None`
explicitly as its own third outcome — `evaluation_failed`, distinct from both "passed" and "flagged" —
log it as its own category, and alert separately if the *rate* of `None`-scores itself spikes, since a
sudden rise in un-scoreable responses is itself a signal (very plausibly a retrieval-layer outage, which
is arguably more urgent than a faithfulness dip). *What would have caught this earlier:* a unit test
against the aggregation function specifically constructed with an empty-context fixture, asserting the
function's behavior on `None` explicitly, rather than only exercising the happy-path scoring cases.

**3. A per-client row-level-security bug in the shared engineering workspace briefly showed one bank's
flagged conversations to the other's reviewers.** Chapter 05 already argues resource-level separation
(separate workspaces per client) should carry the real isolation guarantee, with row-level security as
defense-in-depth only inside the one shared cross-cutting engineering workspace. The gap surfaced exactly
there: a schema migration introduced a new `flagged_conversations` table (replacing an older
`metrics_daily`-derived view) and the RLS rule that filtered the old table was never carried over to the
new one — the report author who ran the migration didn't realize RLS rules don't automatically follow a
table rename/replace. For roughly a day, the "Flagged Case Review" drill-down in the shared workspace
returned rows across both `client_id`s to whichever engineer queried it, meaning a Bank of America
reviewer with access to that shared view could, in principle, have pulled up an HSBC conversation ID.
It was caught when someone noticed the row count in the shared view looked too high relative to either
bank's own traffic in isolation and escalated it as a suspected cross-tenant issue rather than dismissing
it as a display glitch. *Fix:* make the workspace **deny-by-default** — any new table is inaccessible
until an explicit RLS rule is attached, rather than accessible-by-default until someone remembers to add
one — and add an automated check to the report-publishing pipeline that fails the build if a new
queryable table lacks an attached RLS rule. *What would have caught this earlier:* the same deny-by-default
change, applied proactively, plus treating "new table added to the shared workspace" as a checklist item
requiring an explicit RLS sign-off, the same way a schema migration touching production data would
require a review step in a properly gated pipeline.

**4. A race condition double-counted the same flagged conversation, inflating the flagged rate feed into
alerting.** As described in the concurrency section above: the synchronous Tier 1 PII gate and the
asynchronous Tier 3 LLM-as-judge harmfulness classifier both flagged the same `response_id` within the
same aggregation window, and a naive application-level `read current count, write count + 1` pattern
lost one of the two increments to the race, but — counterintuitively — in a small number of observed
cases the *opposite* happened: both increments landed, double-counting a single flagged conversation as
two, quietly inflating the reported flagged rate. This fed directly into false alarms of the same shape
as bug 1. *Fix:* replace the application-level read-modify-write with an atomic SQL increment
(`UPDATE metrics_daily SET flagged_count = flagged_count + 1 WHERE client_id = ? AND day = ?`), and,
better still, stop incrementing a counter at all — instead count `SELECT COUNT(DISTINCT response_id)`
from a `flags` table with a unique constraint on `(response_id, evaluator_name)`, so the same evaluator
can never double-flag the same response even under a retried write, and two *different* evaluators
flagging the same response correctly counts as one flagged conversation, not two. *What would have caught
this earlier:* a concurrency test that fires both evaluators against the same `response_id` genuinely
concurrently (separate threads/processes racing, not a sequential unit test that never exercises the
interleaving) and asserts the final distinct-flagged-conversation count.

The common thread across all four, worth stating as the takeaway: **three of the four (1, 2, 4) are bugs
in the aggregation/alerting layer, not in any individual evaluator's scoring logic** — every evaluator in
these scenarios did its job correctly in isolation; the failures were in how results were combined,
counted, and thresholded afterward. That's a useful, specific answer to "where do bugs actually hide in
a monitoring pipeline" — not in the scoring models themselves, which get the most design attention, but
in the comparatively under-tested glue code that turns individual scores into an alert.

## Concrete timeout, retry, and threshold parameters, and why

- **Tier 1 synchronous evaluation budget: 50ms**, matching the number this course's own system-design
  answer (chapter 01, `99-Interview-QA.md` Q15) already commits to. Chosen because it needs to be a
  negligible fraction of the assistant's own response latency budget — if Tier 1 evaluation cost is
  comparable to the assistant's think time, moderation stops being "free" and starts being the thing
  users notice.
- **LLM-as-judge call timeout: 20 seconds, with 2 retries on transient failure (1s then 4s backoff)**
  before the call is marked `evaluation_failed` rather than retried indefinitely. Twenty seconds is
  generous relative to a typical judge-call latency specifically because these calls are async and
  sampled (chapter 01) — there's no user waiting on them, so the cost of being patient is compute time,
  not user-facing latency, and it's cheaper to occasionally wait out a slow provider response than to
  discard and re-score a sample that would have succeeded on a slightly longer timeout.
- **Sampling rate for LLM-as-judge and Ragas metrics: 15% of live traffic**, plus 100% of a fixed nightly
  regression set (chapter 01's tiered design, given a concrete number here) — chosen as a middle point
  between statistical usefulness (15% of a few-thousand-daily-conversation volume is still hundreds of
  scored samples per day per client, enough for a meaningful rolling trend) and cost (running every LLM-
  judge metric on 100% of live traffic at typical daily volumes would multiply the assistant's own
  inference spend several times over for evaluation alone).
- **Rolling baseline window: 30 days trailing, alert threshold at 2 standard deviations** (chapter 05),
  with the `LEARNING`-phase floor from chapter 06 of **500 responses or 48 hours, whichever comes first**
  before a new assistant-version baseline is eligible to go `ACTIVE`. The 48-hour floor exists
  specifically so a low-traffic client's new version isn't stuck in `LEARNING` indefinitely waiting to
  hit a pure sample-count floor.
- **Minimum sample-count gate before a percentage-based alert can fire: n >= 50** in the evaluation
  window (the direct fix from bug narrative 1) — below that floor, alerting falls back to an absolute
  count threshold instead of a percentage, specifically to prevent a handful of borderline scores during
  low-traffic periods from producing a statistically meaningless but alarming-looking percentage.
- **Metrics-store database connection pool recycle: 1800 seconds (30 minutes).** The hourly aggregation
  job and the alerting checker both hold connections that are frequently idle overnight and around
  low-traffic periods (mirroring exactly the reasoning Course 03 chapter 06 gives for its own
  `pool_recycle=1500` setting on Azure SQL) — a slightly longer recycle window here reflects that this
  metrics-store workload is bursty around the hourly rollup boundary rather than continuously active,
  so connections sit idle longer between real use than a live request-serving API's pool would.
- **Metrics-store write retry policy: 3 attempts with exponential backoff (250ms, 1s, 4s)** for
  transient Azure SQL throttling errors (a real, common behavior under DTU/vCore-based throttling during
  a burst of hourly-rollup writes) — capped at 3 rather than retried indefinitely, so a genuinely down
  database fails the aggregation cycle loudly (logged, retried next scheduled run per the error-handling
  table above) instead of a worker silently blocking on an unbounded retry loop.

## The hardening gap, named candidly

**The alert-routing webhook URL is a bare, unauthenticated incoming-webhook URL, stored as a plain
configuration value rather than in Key Vault.** Incoming webhook URLs for common chat-alert integrations
(Teams, Slack-style) are effectively bearer tokens — anyone who has the URL can `POST` a message to the
channel it targets, with no additional authentication required by the receiving side. If this URL sits
in plaintext application configuration rather than a secret store, anyone with read access to that
configuration (a broader group than anyone with access to actually post real alerts should be) can
either **read where a bank's model-risk on-call alerts go** or, worse, **post a forged alert directly
into a bank's on-call channel** — a fabricated critical-severity page that wastes an on-call engineer's
time at best, or (in the other direction) trains the team to distrust and dismiss real alerts after
enough noise, at worst. The fix mirrors Course 03's session-secret fix exactly in shape: move the webhook
URL to Key Vault (or the equivalent managed-secret store), reference it at runtime rather than reading a
literal string from configuration, and rotate it per client so a compromised URL for one bank's channel
doesn't also expose the other's.

## Two things already done right, worth highlighting positively

- **The metrics-store aggregation job's staging-table-then-atomic-swap pattern.** Writing a full rollup
  to a staging table and only swapping it into the dashboard-facing table on complete success (rather
  than updating rows incrementally in place) means a failed or interrupted rollup run can never leave the
  dashboard showing a half-written, internally inconsistent hour — the worst case is *stale* data, never
  *wrong* data, which is exactly the right failure mode to prefer for a system a risk committee is
  relying on for accuracy.
- **Tier 1's fail-closed-on-error, fail-open-on-timeout asymmetry.** Treating "the safety gate crashed"
  and "the safety gate was slow" as two different failure modes requiring two different responses — one
  needs to be conservative because an unhandled exception is a real correctness gap, the other needs to
  be permissive because a hard latency SLO on a safety check that occasionally runs long shouldn't be
  allowed to take down the whole assistant's availability — is a deliberately-reasoned design, not an
  accidental default, and it's worth stating that reasoning explicitly if asked why the two failure modes
  aren't handled identically.

## Tying It Back

Production-grade, for an evaluation and monitoring platform specifically, means something slightly
different than it does for a request-serving API: the assistant itself keeps working even when the
monitoring layer is fully broken (Tier 1 aside), which is a genuine strength of the tiered design — but
it also means monitoring-layer bugs are quieter and easier to let run for longer, because nothing about
them looks like an outage. Three of this chapter's four illustrative bugs (1, 2, 4) are exactly that
shape: not a crash, not a 500, just a slowly, silently wrong number feeding a dashboard and an alert rule
that a risk committee trusted to be right. That's the specific operational discipline this kind of
system demands beyond the usual timeout/retry/pooling hygiene — auditing the aggregation and alerting
glue code at least as carefully as the evaluators themselves, because that's where all four illustrative
bugs in this chapter actually lived.
