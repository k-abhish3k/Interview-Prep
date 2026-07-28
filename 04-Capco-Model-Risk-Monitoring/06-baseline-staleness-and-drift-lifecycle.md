# 06 — Baseline Staleness and the Drift-Detection Lifecycle

## Why this chapter exists

Chapter 05 already establishes that alerting is **rolling-baseline-based**, not fixed-threshold: "alert
if the 24-hour average deviates more than 2 standard deviations from the trailing 30-day average." That
detail sounds solid in isolation — until a technical interviewer asks the natural follow-up: **"What
happens to that 30-day trailing baseline when the thing you're monitoring changes on purpose — a new
system prompt, a new model deployment, a new retrieval index? Does the baseline know that happened, or
does it just keep averaging like nothing did?"**

That question is this chapter, worked all the way through. As with the rest of this course, there is no
real source repository backing these specifics — this is a plausible, technically detailed
reconstruction of how a Consultant-II-built monitoring platform would plausibly behave and what a
thoughtful engineer on this project would propose to fix, not a verified account of Capco's actual
implementation. Present it in an interview as "here's the design I'd defend," not as recalled fact.

The honest answer has the same five-part shape as Course 05's document-versioning chapter: what a
realistic implementation does today, a manual workaround that papers over the gap, two concerns that
are easy to conflate and shouldn't be, a genuine proposed fix, and a forward-looking tie to the system
that would need to change upstream for the fix to actually close.

## Part 1 — What a realistic implementation does today: the baseline doesn't know the assistant changed

Chapter 05 already names two pieces that exist independently of each other, and the gap lives exactly in
the seam between them:

1. **The rolling baseline** — an unweighted trailing statistic (mean and standard deviation) computed
   directly from the raw per-response metric history in the metrics store, over the last 30 days, for a
   given `client_id` and metric (faithfulness, harmfulness-flag rate, latency p95, and so on). This is
   pure time-series math: every row in the window counts equally, regardless of which prompt version,
   model deployment, or retrieval index produced it.
2. **Deployment markers** — the "clear visual marker for deployments/prompt changes" chapter 05 already
   describes on the executive trend view, so a human glancing at the chart can eyeball "the dip started
   right after that vertical line."

The gap: **the deployment marker is a chart annotation, not an input to the alerting engine.** It's
metadata written to a timeline-annotations table (typically by whoever triggers the release — a manual
dashboard entry, or a webhook from the CI/CD pipeline in Course 06) purely for a human reviewer's benefit.
The rolling-baseline calculation described in chapter 05 reads directly from the raw metric history and
has no join, filter, or weighting against that annotation table at all. Concretely, that means:

- If Course 01's assistant ships a **legitimately better** prompt that changes response style — say,
  more concise answers that happen to also score slightly lower on Ragas answer relevancy purely because
  they're shorter, not because they're worse — the trailing 30-day baseline is still dominated by 29
  days of pre-change data. The new, intentionally-different style looks like *drift* relative to a
  baseline that has no idea a deliberate change happened, and the on-call rotation gets paged for a
  "regression" that is, in fact, a shipped improvement.
- If Course 01's assistant ships a **genuinely worse** change — a retrieval index pointed at a stale
  snapshot, a system prompt regression — the opposite failure happens. Because the baseline is a
  *trailing* window, each additional day of degraded post-change data pulls the rolling mean and standard
  deviation toward treating the new, worse behavior as "normal." A real regression can take up to 30 days
  to fully "roll out" of its own contaminated baseline, during which the alert threshold is quietly
  loosening around exactly the failure it should be catching.

Both directions are the same root cause: **the baseline has no concept of a version boundary.** It
treats every day in the trailing window as equally representative of "the current assistant," which is
only true between deploys and false across them.

A related, narrower manifestation worth naming in the same breath: any **non-LLM allow-list of
"known-good" answer patterns** (chapter 01's cheap deterministic Tier 1 checks — e.g., a regex or
template match confirming a response contains the currently-expected disclaimer phrasing for a given
intent) goes stale the exact same way. If Course 01's assistant's prompt is rewritten and the canonical
disclaimer wording changes from "This is not financial advice" to "This information is educational and
not a substitute for financial advice," a Tier 1 allow-list still keyed to the old exact phrasing starts
flagging every correctly-updated response as *missing* the required disclaimer — a false-positive storm
caused by the assistant getting better, not worse. It's the same "comparison point didn't move when the
target did" failure as the rolling baseline, just implemented as a string match instead of a mean/std
calculation.

## Part 2 — The manual workaround, achievable today with existing pieces

Nothing about this gap is unfixable with what already exists — it's coordination-by-convention, not a
missing capability:

1. **Silence alerting for a known deploy window.** A per-`client_id` "maintenance window" flag on the
   alert-rules table (chapter 05's alerting layer) that a human on the GenAI team sets manually before a
   known prompt/model/retrieval change ships, suppressing paging (but not logging) for, say, 24-48 hours.
2. **Manually reset the baseline's start date after the window closes.** Rather than waiting up to 30
   days for pre-change data to naturally roll out of the trailing window, someone manually truncates the
   baseline calculation's lookback to start from the deploy date, so the "trailing 30 days" becomes
   "trailing 30 days, or since the last known deploy, whichever is shorter."
3. **Manually refresh the Tier 1 allow-list.** Whoever owns the prompt change also has to remember to
   update the disclaimer/known-good-pattern regex in the same pull request, so the two don't drift apart.

This is a real, working pattern — but every step depends on a human **knowing** a change is about to
ship and **remembering** to flip the flag, update the regex, and reset the baseline, in that order,
before or immediately after the deploy. Nothing in the monitoring platform enforces it, and nothing stops
a hotfix, a config-only change, or a model-provider-side update (see Part 3) from shipping without anyone
touching the maintenance-window flag at all. That's the precise, honest statement of the gap — not "we
have no way to handle this," but "the way we handle this today is entirely manual coordination, and nothing
technical prevents someone from skipping it."

## Part 3 — Two concerns that are easy to conflate, and shouldn't be

**Genuine metric drift** — the assistant's behavior degrades against a **fixed** target, for reasons
that have nothing to do with an intentional change to the assistant itself:

- The knowledge base underlying retrieval goes stale — a policy document referenced by the assistant's
  answers ages past its effective date without anyone updating the index, so completeness/faithfulness
  scores genuinely, correctly decline.
- The population of user questions shifts (a seasonal spike in a topic the assistant was never well
  tuned for) and quality on that segment genuinely is worse, not just differently distributed.
- An upstream dependency degrades — Azure OpenAI's underlying model deployment experiences elevated
  latency, or a **model-provider-side silent update** changes the base model's behavior without Course 01's team changing a single line of prompt or config (this happens in practice with hosted models more
  often than teams expect, and it's the sharpest version of this problem, because *nobody on this side of
  the fence even triggered a deploy* — there is no deployment marker to correlate against at all).

This is exactly what the rolling baseline should be catching, and a real regression here **should** move
the baseline — that's the system working as intended.

**Baseline staleness** — the comparison point itself is wrong, because the thing being measured
legitimately changed underneath it: a new system prompt, a new model deployment name, a new retrieval
index version. The "drift" isn't a change in behavior relative to a stable target — it's an artifact of
comparing new-target metrics against an old-target baseline. Nothing about the assistant is actually
misbehaving; the *measurement setup* hasn't caught up.

The reason it's worth stating this distinction explicitly, unprompted, in an interview: conflating them
leads to exactly the wrong operational response in both directions. Treating baseline staleness as
genuine drift means someone burns an on-call rotation's afternoon "root-causing" a regression that's
actually a shipped improvement. Treating genuine drift as baseline staleness — assuming "oh, someone must
have deployed something" and resetting the baseline to absorb it — silently launders a real regression
into the new normal, which is the worse of the two mistakes, because it makes the monitoring platform
complicit in hiding the exact failure it exists to catch. The model-provider-side silent-update case above
is the sharpest illustration of why this distinction matters: it looks identical to baseline staleness
(metrics shift right after some kind of external change) but the correct response is the genuine-drift
response (investigate and likely alert), not the baseline-reset response, because nobody on the
monitored side intentionally revised anything.

## Part 4 — A proposed design: baseline versioning tied to an assistant version tag

This is explicitly a **proposal**, not a description of anything implemented — labeled as such
throughout, in keeping with this course's honesty conventions.

**Step 1 — the assistant needs to tag itself.** Course 01's deployed assistant computes a deterministic
`assistant_version_tag` — e.g., a short hash over the combination of the active system prompt text, the
model deployment name/version, and the retrieval index name/version:

```python
# Proposed addition to Course 01's assistant — not implemented today
import hashlib

def compute_assistant_version_tag(system_prompt: str, model_deployment: str, retrieval_index: str) -> str:
    fingerprint = f"{system_prompt}|{model_deployment}|{retrieval_index}"
    return hashlib.sha256(fingerprint.encode("utf-8")).hexdigest()[:12]
```

That tag gets attached to **every** per-response metric record chapter 05 already defines — one new
field, `assistant_version_tag`, alongside `client_id` and the rest of the schema. This is the single
change that makes everything downstream possible: without it, the monitoring platform has no reliable
signal that a revision happened at all, and is stuck inferring it indirectly from a deployment-marker
annotation someone may or may not have remembered to log.

**Step 2 — the monitoring platform keys baselines by `(client_id, assistant_version_tag)`, not just
`client_id`.** A new table, proposed:

```python
# Proposed additions — not implemented today
baseline_id = sqla.Column(sqla.String(36), primary_key=True)
client_id = sqla.Column(sqla.String, nullable=False)
metric_name = sqla.Column(sqla.String, nullable=False)          # e.g. "faithfulness", "harmfulness_flag_rate"
assistant_version_tag = sqla.Column(sqla.String(12), nullable=False)
status = sqla.Column(sqla.String, nullable=False)               # LEARNING | ACTIVE | RETIRED
sample_count = sqla.Column(sqla.Integer, default=0)
rolling_mean = sqla.Column(sqla.Float, nullable=True)
rolling_std = sqla.Column(sqla.Float, nullable=True)
activated_at = sqla.Column(sqla.DateTime, nullable=True)
superseded_baseline_id = sqla.Column(sqla.String(36), nullable=True)
# The baseline (for this client + metric) that this one replaced, if any —
# kept, not deleted, so a "did the new version actually help" comparison
# report stays possible after the switch.
```

**Step 3 — the lifecycle.** The first time a response bearing a previously-unseen
`assistant_version_tag` arrives for a given `client_id`, the platform opens a new baseline row in
`LEARNING` status. While `LEARNING`:

- Alerting for that client/metric continues comparing against the **previous** version's `ACTIVE`
  baseline — the system does *not* go silent, which matters, because a genuine regression shipped in the
  same release that introduced the new tag still needs to be catchable during the transition, not
  invisible for the sake of avoiding false positives.
- Any breach detected during this window is still raised, but tagged distinctly (e.g.,
  `"post-deploy, new baseline stabilizing"`) rather than paged with the same urgency as a breach against
  a fully-`ACTIVE` baseline — surfacing it on a digest rather than waking someone at 2 a.m., since it's
  genuinely less certain whether it's real drift or an artifact of comparing across the version
  boundary.
- The `LEARNING` baseline accumulates its own `rolling_mean`/`rolling_std` in the background, in fixed
  chunks (e.g., every 100 responses), and is considered **stabilized** — ready to flip to `ACTIVE` — once
  two conditions both hold: (a) it has accumulated at least a floor number of samples (proposed: 500
  responses, or 48 hours elapsed, whichever comes first, mirroring the volume-aware thresholding chapter
  05 already argues for elsewhere), and (b) the chunk-over-chunk change in its own rolling mean has
  fallen below a small tolerance for two consecutive chunks (a simple stand-in for a proper statistical
  process-control stability check, without requiring a full changepoint-detection library).
- On stabilization, the new baseline flips to `ACTIVE`, the previous `ACTIVE` baseline flips to
  `RETIRED` (with `superseded_baseline_id` pointing at it, not deleted), and alerting for that client/
  metric switches to comparing against the new, correctly-scoped baseline.

`notebooks/05_baseline_drift_and_versioning_simulation.ipynb` implements this exact lifecycle end to end
on synthetic metric streams — a stable baseline, a genuine-drift scenario, and a version-change scenario
— and shows the stabilization check triggering the `LEARNING -> ACTIVE` transition on simulated data.

**Why keep the retired baseline instead of deleting it:** the moment a new version stabilizes, a
genuinely useful question becomes answerable for the first time — "did version B actually score better
than version A on faithfulness, once both had a fair, correctly-scoped comparison window?" — which is a
real product/risk-committee question ("did that prompt change help or hurt") that a purely trailing,
version-blind baseline can never answer cleanly, because it never had a clean before/after boundary to
begin with.

## Part 5 — The forward-looking tie: this needs to be a CI/CD event, not a metrics-side inference

The weakest link in Part 4's design is where `assistant_version_tag` actually comes from. If the
monitoring platform tries to *infer* "something changed" purely by watching for a shift in the metric
distribution, that's circular — a distribution shift is exactly what it's trying to classify as either
genuine drift or a version change, so it can't also be the signal that decides which one it is.

The fix has to live upstream, in Course 06's release pipeline: the Azure DevOps YAML pipeline that
deploys Course 01's assistant should be the **one true source** of "a revision happened," emitting the
version tag as a first-class release-gate step — computing the hash over the deployed prompt/model/
retrieval-index configuration and writing it (via a webhook, or as a custom property on the deployed App
Service's Application Insights telemetry) at the moment of deployment, not reconstructed after the fact
by the monitoring side guessing from metric shifts. That closes the loop system-wide: **Course 06's
pipeline tags the release, Course 01's assistant stamps that tag on every response it produces, and
Course 04's monitoring platform keys its baselines to that tag** — three separate courses' systems each
doing the one thing only they can reliably do, instead of the monitoring platform trying to reverse-engineer
a deploy event from noisy downstream statistics. That's the same shape of cross-system dependency Course 05's versioning chapter names for its own domain (this service can't fix HEXA's missing version concept
unilaterally) — the pattern generalizes: a monitoring or lifecycle fix that lives entirely inside one
system's database is necessarily incomplete if the signal it needs to be correct is only reliably
producible by a different system upstream.

## The full interview-ready answer, assembled

> "Today, the rolling baseline chapter — a 30-day trailing mean and standard deviation — has no concept
> of a version boundary. There's a deployment marker on the trend chart, but it's a chart annotation for
> a human to eyeball, not an input the alerting engine actually reads. So a deliberate prompt or model
> change either looks like a false-positive regression for weeks, because the trailing baseline is still
> mostly pre-change data, or a genuine regression slowly gets absorbed into the baseline as 'normal' over
> the same trailing window, which is the worse failure mode. The workaround that exists today is entirely
> manual — silence alerting for a known deploy window, then manually reset the baseline's lookback — and
> it only works if whoever's shipping the change remembers to do both steps. It's also worth being
> precise that this is a different problem from genuine drift: a stale knowledge base, a shifting user
> population, or even a silent model-provider-side update are real degradations the baseline *should*
> catch, and conflating that with 'someone must have deployed something' and resetting the baseline in
> response would quietly launder a real regression into the new normal. The fix I'd propose is baseline
> versioning: tag every response with a version fingerprint of the active prompt/model/retrieval config,
> key each client's baseline to `(client_id, assistant_version_tag)`, run new versions through a
> `LEARNING` phase where alerts still fire but at lower urgency until the new baseline stabilizes, then
> flip it to `ACTIVE` and retire — not delete — the old one so a clean before/after comparison stays
> possible. But that version tag can't be inferred reliably from the metrics side — it has to come from
> the CI/CD pipeline that actually deploys the assistant, as a release-gate event, not reconstructed
> after the fact from a distribution shift that's ambiguous by construction."

## Tying It Back

The strongest part of this answer isn't the proposed `baseline_versions` schema in Part 4 — it's Part 3:
naming, unprompted, the exact two failure modes a naive fix could introduce (treating a shipped
improvement as a regression, or treating a real regression as an artifact of a deploy that quietly gets
absorbed as normal) and showing the model-provider-silent-update edge case where the two are hardest to
tell apart. Just as Course 05's versioning chapter earns credibility from precisely stating what's
*not* built, this chapter earns it from precisely stating *which two things not to confuse* — that's
the detail that separates "I'd add a version column" from "I understand exactly what goes wrong if the
version column isn't backed by a real, externally-triggered signal."
