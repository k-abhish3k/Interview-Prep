# 07 — Production Resilience and Operational Engineering

## Why this chapter exists

This is the "not limited to architecture diagrams, cover production grade" chapter for the CI/CD
platform itself — the same spirit as course 03's chapter 06, applied to the pipeline rather than to a
single service. Chapter 06 covered what happens when three independently-versioned services drift out
of compatibility; this chapter covers the pipeline's own failure modes: exactly what happens when each
*stage* fails, the concurrency traps that only show up under real operational load, four concrete "bug
found and fixed" narratives specific to CI/CD mechanics, the actual timeout/retry values worth setting
and why, and one candidly-named hardening gap worth discussing honestly rather than glossing over.

**The same honesty constraint as chapter 06 applies here, and it's worth restating plainly:** there is
no real proprietary pipeline repository backing this course. Every table, YAML fragment, and bug story
below is a plausible, technically detailed **illustrative reconstruction** of the kind of failure a
multi-client Azure DevOps + Terraform pipeline realistically produces — grounded in how these tools
actually behave, not a claim about a specific incident that happened on this engagement. Treat it as a
strong, defensible system-design-and-postmortem answer, not a verified fact.

## The pipeline failure-and-rollback decision table

What a well-built pipeline does — and what a human has to decide — at each stage when something goes
wrong:

| Stage | Failure | What happens automatically | Rollback/recovery decision |
|---|---|---|---|
| Build/lint | Dependency install fails, or lint finds a blocking error | Pipeline stops immediately; no artifact produced, no environment touched | None needed — nothing was deployed. Fix the code, re-run. Cheapest possible failure. |
| Unit/integration test | A test fails | Pipeline stops after the Build stage; no image is published to the registry | Same as above — the failure is fully contained before anything reaches an environment. |
| Security/dependency scan | A critical CVE is found in a dependency or base image | Image is built but the pipeline should **not** auto-promote it past dev, even though the image technically exists in the registry | Human triage: patch the dependency and re-run, or — if genuinely low-risk and time-critical — an explicit, logged override by someone with authority to accept the risk. Never a silent auto-promote. |
| `terraform plan` | Plan fails outright (bad provider config, expired credentials) | Stage fails; no `apply` is attempted | Fix the plan input and re-run; nothing changed in real infrastructure. |
| `terraform plan` | Plan succeeds but shows an unexpected `destroy` on a stateful resource (a database, a Key Vault holding live secrets) | Plan output is posted for review; nothing applies automatically | **This is a human judgment call, and it's the single highest-leverage moment to catch a bad change** — see the hardening gap below for why this is currently enforced by reviewer diligence rather than policy. |
| `terraform apply` | Apply fails partway (one resource created, the next fails) | Terraform is not transactional — state now reflects a **partial** success; the stage fails | Re-running `apply` after fixing the root cause is usually safe, because Terraform reconciles from whatever state now exists rather than starting over blind. **Never** manually edit or delete the state file to "unstick" it without understanding exactly what state it's actually in — that's the fastest way to turn a partial failure into an unrecoverable one. |
| `terraform apply` | Apply succeeds but computes an unexpected **no-op** because of drift (see bug narrative 2 below) | Pipeline reports success; the intended change silently did not take effect | This is the most dangerous failure in the whole table because it looks identical to success — caught only by a subsequent `terraform plan` against real infrastructure, or by the smoke test catching the *symptom* of the change not having applied. |
| Deploy (slot deploy) | Container fails to start in the staging slot | The deploy task fails; the slot swap that promotes it to production never happens | No user impact yet — production traffic never touched the broken container. Fix and redeploy to the slot; retry is cheap because nothing user-facing changed. |
| Deploy | Slot deploy succeeds, slot swap completes, but the **post-swap** smoke test fails | Pipeline marks the release failed | **Slot swap back immediately** (chapter 04) — this is the fastest available recovery and should be the default first response, not a debugging session while users are on the broken version. |
| Smoke test | Smoke test passes, but it was checking the wrong thing (bug narrative 3 below) | Pipeline reports a clean, green release | No automatic recovery is possible here, by definition — a false-green smoke test means the pipeline itself believes everything is fine. The real signal arrives later, via monitoring/alerting or a user report, which is exactly why smoke-test correctness has to be reviewed with the same rigor as the application code it's testing. |

The one pattern worth naming explicitly across this whole table: **the earlier in the pipeline a
failure occurs, the cheaper and more automatic the recovery is** — a build failure costs nothing but a
re-run; a false-green smoke test costs an undetected production incident. That's the entire argument for
investing disproportionately in the earliest, cheapest gates (linting, unit tests) rather than treating
them as a formality on the way to the "real" deploy stage.

## Concurrency caveat: state-lock contention and the shared release manifest

Chapter 03 established that Terraform state is kept in a remote, locked backend, with a separate state
file key per client and per service (`"<client>/<service>/<env>.tfstate"`) specifically so HSBC's and
Bank of America's `apply` runs can never contend for the same lock. That isolation holds for the common
case — but two realistic scenarios still produce genuine lock contention, worth being able to describe
precisely rather than hand-waving "state locking prevents conflicts" as if it eliminates every race:

- **A scheduled drift-detection run (chapter 02's nightly `terraform plan` job) overlapping with an
  emergency hotfix `apply` for the same client and service.** Both target the identical state file. The
  scheduled job only runs `plan` (read-only against state, no lock needed for `plan` on most backends,
  though some configurations still acquire a read lock) — but if the drift job is itself configured to
  auto-`apply` a detected drift (a debatable but real pattern for keeping environments self-healing), it
  can hold the write lock at exactly the moment an on-call engineer is trying to push a genuine
  emergency fix through the same pipeline. The engineer's `apply` fails fast with a "state is locked"
  error rather than corrupting anything — but during an active incident, a confusing lock error is
  exactly the kind of friction that costs minutes an incident doesn't have. **The mitigation:** scheduled
  drift-detection jobs should default to plan-only with a human-reviewed apply, never auto-apply, precisely
  because an emergency hotfix window is unpredictable and a self-healing job with its own apply step is
  an avoidable second writer.
- **The shared release-manifest store from chapter 06** (the platform-level artifact recording which
  triple of service versions is currently live per client) is, by construction, written by three
  independently-scheduled pipelines — the chatbot's, monitoring's, and uploader's platform-release
  stages. If two of those three happen to deploy or roll back within the same short window (plausible
  during a coordinated release, or during an incident where two services are being fixed at once), both
  platform-release stages attempt to read-modify-write the same manifest. Without an explicit
  concurrency control on the manifest store itself (an optimistic-concurrency check — read the current
  `manifest_version`, write only if it hasn't changed since, otherwise re-read and retry — the same
  pattern SQL-based optimistic locking uses elsewhere in this curriculum), the second writer can silently
  overwrite the first writer's update with a manifest computed from stale data, which is precisely the
  kind of bug the manifest was built to prevent in the first place. **The mitigation is the same
  pattern as Terraform's own state lock:** version the manifest, and make every write conditional on the
  version it read matching the version it's about to overwrite.

A related, narrower caveat worth naming if asked specifically about authentication: a pipeline stage
that resolves and caches a service-connection's authentication token once at stage start can find that
token stale mid-run if the underlying service-principal secret is rotated (deliberately, as part of a
scheduled rotation, or urgently, in response to a suspected leak) while a long-running multi-stage
deploy is already in flight. The practical effect is a deploy that authenticates successfully in an
early stage and then fails with an authorization error in a later one, which reads confusingly (why did
it work five minutes ago?) unless you know to check whether a rotation happened in between. The
mitigation is to keep individual pipeline runs short enough that this window is small, and to treat any
mid-run auth failure as a first-class signal to check "did a secret rotate in the last hour" before
assuming the credential itself is wrong.

## Four real-shaped bugs, illustrative but concrete

Framed the same way as course 03's chapter 06 — genuinely specific, plausible bugs that a multi-client
Azure DevOps + Terraform pipeline of this shape produces, good material for "tell me about a bug you
found" questions, each with what practice would have caught it earlier:

**1. A copy-pasted template parameter leaked HSBC's variable group into Bank of America's deploy
stage.** Chapter 02's per-client stage matrix works by instantiating the same `deploy-to-client.yml`
template twice, once per client, with only the parameter values differing. The realistic version of
this bug: a new client-specific setting (say, a feature flag controlling an HSBC-only compliance
report) was added by copying the `DeployHSBC` stage block, renaming it to `DeployBofA`, and updating the
obvious parameters (`serviceConnection`, `webAppName`) — but the copy-paste missed updating
`variableGroup`, which was left pointing at `hsbc-prod-secrets`. The `DeployBofA` stage then deployed
correctly to Bank of America's App Service and its own service connection, but with **HSBC's** Key
Vault URI and connection strings injected as app settings — meaning Bank of America's production
container briefly ran with credentials scoped to HSBC's resources. *What would have caught this
earlier:* an automated post-deploy assertion (chapter 04's Q17 pattern, "confirm the deploy stage's
service connection and variable group actually resolved to the expected client's resources") run as a
required check before a client's deploy stage is considered complete — checking, for example, that the
Key Vault URI injected into Bank of America's app settings actually points at a Key Vault whose name
contains `bofa`, not `hsbc`. This is exactly the class of bug where the pipeline itself reports success
(both deploys "succeeded") and only a cross-check against expected identity, not just expected outcome,
catches it.

**2. A `terraform apply` that silently no-op'd because of state drift from an out-of-band manual
change.** An engineer, under time pressure during an earlier incident, changed an App Service
Application Setting directly in the Azure Portal rather than through Terraform — a reasonable-feeling
shortcut in the moment, and exactly the kind of thing chapter 03's "why IaC, specifically" section warns
against. The next scheduled pipeline run's `terraform apply` was configured with `-refresh=false` (added
at some point to speed up routine applies, since a full refresh against every resource adds real time on
a large module), which meant it never re-queried the actual live value of that setting — it computed its
diff purely against the **last recorded state**, which still reflected the pre-incident value, matched
the desired config, and concluded "no changes needed." The apply reported a clean success. The
Portal-made change was never reverted, and the *next* legitimate Terraform-managed change to that same
setting silently failed to take effect either, because Terraform still believed the setting already
matched. *What would have caught this earlier:* never running `apply` with `-refresh=false` against a
shared production-adjacent state — the time saved is real but small, and the failure mode it opens
(silent divergence between believed and actual state) is exactly the kind of bug that's invisible until
something else depends on the drifted value. A scheduled, refresh-enabled `plan` (chapter 02's nightly
drift-detection job, run *without* the refresh-skipping flag) would have surfaced the drift the very
next night as an unexpected diff, rather than it persisting silently for weeks.

**3. A smoke test that always passed because it checked the wrong endpoint after a URL refactor.**
`scripts/smoke_test.py` (chapter 02) was originally written to hit `<app-url>/health`. When the
document-uploader service (course 03) was refactored to sit behind an API-Gateway-style path prefix
(routes moved under `/api/v1/...`, with the bare root path left in place only to avoid breaking an old
health-check convention), the actual functional health check moved to `/api/v1/health`, but the smoke
test's hardcoded URL was never updated. The bare `/` route, left behind for compatibility, happened to
return a generic `200 OK` landing-page response rather than a `404` — so the smoke test kept passing on
every single deploy, for months, without ever exercising the real, refactored endpoint. Every actual
regression in the health path during that window would have shipped to production with a fully green
pipeline. *What would have caught this earlier:* a smoke test that asserts on the **response body
shape**, not just the HTTP status code — e.g., confirming the health response contains an expected field
like `{"status": "ok", "service": "document-uploader"}` rather than accepting any `200`. A generic
landing page returning `200` with no matching body would have failed that stronger assertion
immediately, on the very first deploy after the refactor, instead of silently passing for months.

**4. A rollback that restored the app code but not a corresponding database migration, causing a schema
mismatch.** The document-uploader service shipped a migration renaming a column the IWPB workflow relied
on (the kind of schema change chapter 03/course 03 would apply via an Alembic-style migration step run
ahead of the new image's deploy). A subsequent, unrelated bug in that same release forced a rollback —
executed the fast, routine way (chapter 04): redeploy the previous, immutable image tag. That correctly
restored the *old application code*, which referenced the *old* column name — but the database migration
that renamed the column had already run and was never reverted, because the rollback pipeline's scope
was "redeploy the previous container image," not "also run the corresponding down-migration." The
restored old code immediately started throwing errors on every query referencing the now-renamed
column. *What would have caught this earlier:* treating a schema-changing migration as a change that
requires its own, explicit rollback plan (a tested down-migration) recorded and validated *at the time
the forward migration ships*, not improvised during an incident — and, more fundamentally, recognizing
that "rollback" for a service with a stateful schema is never simply "redeploy the previous image tag"
the way it is for a stateless service; it's two coordinated reversals (code and schema) that have to be
sequenced correctly, or not attempted as a combined "just roll back" instruction at all.

The common thread across all four, worth stating as the takeaway: **three of the four look identical to
success from the pipeline's own point of view** — the copy-paste leak, the silent no-op apply, and the
false-green smoke test all reported clean, green pipeline runs. Only the fourth (the migration mismatch)
announced itself immediately, and only because the application crashed loudly rather than degrading
silently. That asymmetry is exactly why smoke-test *content* (what it actually asserts, not just whether
it runs) and post-deploy identity checks deserve as much engineering attention as the deploy mechanics
themselves.

## Concrete timeout and retry values, and why they matter

Specific, defensible numbers worth being able to state rather than a vague "we set some timeouts":

- **Pipeline job timeout: 60 minutes default, explicitly reduced to 20 minutes for the Build/Test stage
  and left at 60 for Deploy stages.** Azure DevOps's platform default (60 minutes on Microsoft-hosted
  agents for many project tiers) is generous enough to mask a genuinely hung step — a test suite stuck
  in a deadlock, a `pip install` stalled on a slow mirror — for far longer than useful. Tightening the
  Build/Test stage's timeout to 20 minutes means a hang there fails fast and visibly instead of quietly
  consuming an hour of agent time before anyone notices; the Deploy stage keeps the longer default
  because a slot deploy plus warm-up plus smoke test genuinely can take longer, especially for a
  cold-starting container.
- **`terraform apply` timeout: 30 minutes**, set explicitly on the pipeline task (not left at whatever
  the task's own default is) — long enough for a full App Service + Key Vault + networking module to
  provision from scratch (the slowest realistic case, e.g. standing up a brand-new client environment),
  short enough that a genuinely stuck `apply` (waiting on a resource that will never become healthy,
  such as a bad DNS/networking dependency) fails and pages someone rather than blocking the pipeline
  queue indefinitely.
- **Smoke test HTTP request timeout: 10 seconds per request, with 3 retries and a 5-second backoff
  between attempts** — a `requests`-based smoke test call needs a bounded timeout so a slow-to-respond
  (not necessarily broken) freshly-swapped app doesn't get falsely marked unhealthy on the very first
  request after a slot swap, when the container may still be finishing its own warm-up; the retry-with-
  backoff absorbs that startup window without absorbing an indefinitely hung one — after 3 failed
  attempts (roughly 30–45 seconds total), the smoke test fails for real rather than retrying forever.
- **Approval-gate timeout: 24 hours** on the `Environment` resource's approval check, after which the
  pending deployment is automatically rejected rather than left open indefinitely — a stale, forgotten
  approval request sitting open for a week is itself a minor security exposure (an old, possibly
  no-longer-relevant change sitting one accidental click away from hitting production) and a rollback
  hygiene problem (nobody remembers why it's still pending), so an explicit expiry forces a deliberate
  re-approval rather than an ambient one.

The underlying principle behind all four: **every timeout value is a deliberate trade-off between "fail
fast enough to notice" and "don't fail on a transient, recoverable delay,"** not a default left
untouched — and being able to state the actual number and the actual reasoning behind it, rather than
"we have timeouts configured," is the difference that shows up under a good follow-up question.

## The manual approval convention: a real, named hardening gap

This is a genuine gap worth naming candidly, in the same spirit as course 03's hardcoded session secret:
**the requirement that a `terraform plan` showing an unexpected `destroy` on a stateful resource must be
manually reviewed before approval is enforced by reviewer diligence and team convention, not by an
automated policy gate.** Nothing in the pipeline as described actually *inspects* the plan's JSON output
and blocks the approval step if it contains a `delete` (or a `create`+`delete` replace) targeting a
resource type known to hold live data — a Key Vault, an Azure SQL database, a storage account. The
approval gate exists (chapter 04), and the plan output is genuinely visible to the approver — but
whether that approver actually reads the plan carefully enough to notice an unexpected destroy, on a
given Tuesday, under time pressure, with three other approvals also waiting, is a question of individual
diligence, not a guarantee the system enforces.

The fix is well-understood and not exotic: an automated policy check (an Open Policy Agent/Conftest
rule, or even a simple script parsing `terraform show -json` output the same way notebook
`02_terraform_basics_demo.ipynb` does) that runs as a required, blocking pipeline stage *before* the
approval gate — failing the stage outright, with no human override short of an explicit, logged
exception process, if the plan contains a destroy on a resource type flagged as stateful. This turns
"the reviewer is supposed to catch this" into "the pipeline cannot proceed past this point regardless of
who's reviewing or how busy they are," which is a materially stronger guarantee. Notebook
`05_terraform_plan_diff_safety_gate.ipynb` implements a version of exactly this check, standalone.

The honest framing worth using if asked directly: *"this wasn't automated because building a policy
engine felt like more upfront investment than the number of actual close calls justified at the time —
but it's a well-known gap, the fix is a single, well-scoped pipeline stage, and I'd prioritize it the
moment 'a reviewer missed an unexpected destroy' stopped being hypothetical and became a real near-miss
or incident."*

## Tying It Back

Production-grade CI/CD isn't "the pipeline never fails" — it's "every failure mode is known, the
recovery path for each one is decided in advance rather than improvised, and the team can say precisely
which failures look like success until something downstream notices, rather than discovering that
gap during an incident." This pipeline's actual profile, stated candidly: the earliest, cheapest gates
(build, lint, test) fail loudly and cost nothing to recover from; the later, more consequential ones
(a silent `apply` no-op, a false-green smoke test, a copy-pasted variable-group leak) can look
identical to success, which is exactly why smoke-test content, post-deploy identity checks, and a
scheduled refresh-enabled drift check matter as much as the deploy mechanics themselves; two genuine
concurrency traps (state-lock contention between a scheduled drift job and an emergency hotfix, and an
un-versioned shared release-manifest write) are real and have concrete, well-understood mitigations; and
one hardening gap — policy-as-code for destructive Terraform plans — is named honestly as a known,
prioritizable gap rather than glossed over as already solved.
