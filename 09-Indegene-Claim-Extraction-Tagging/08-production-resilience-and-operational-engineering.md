# 08 — Production Resilience and Operational Engineering

This chapter is the "not limited to architecture diagrams, cover production grade" chapter for the
claims pipeline — a realistic error-handling table (which failures should block publishing vs. just
flag for review), a genuine concurrency/scaling caveat in how the Sagemaker multi-model endpoints serve
two clients, four concrete "bug found and fixed" narratives specific to an NLP classification/compliance
pipeline, the operational settings (thresholds, timeouts, batch sizes) worth being able to discuss with
real numbers, and one candidly-named hardening gap.

As with the rest of this course, none of this is drawn from a real, verified source repository —
everything below is a plausible, technically detailed reconstruction of the kind of production issues a
system shaped like this one would realistically run into, written with the same specificity you'd use if
you'd actually lived through it, but explicitly not a claim about verified Indegene implementation
detail.

## The realistic error-handling table

Adapted to the pipeline stages from the chapter 00 architecture diagram — what a well-behaved
implementation should do at each failure point, and why:

| Failure | What should happen | Why |
|---|---|---|
| Text ingestion/OCR fails to parse the source file (corrupted PDF, unsupported format) | **Block publishing entirely** — return an ingestion error to the submitter | No claims were even extracted; there is nothing to have "passed," so there is nothing safe to auto-approve |
| Claim Extraction model invocation times out or throws | **Block** — content moves to an explicit "extraction failed, needs manual triage" state | The absence of extracted claims must never be interpreted as "no claims found, therefore compliant" |
| Claim Extraction returns zero claims for content well above a length/word-count heuristic | **Flag for review**, not a hard block | Suspicious, not necessarily wrong — a human should sanity-check "clean" results on the rare content this heuristic catches |
| ISI Classifier confidence below a set threshold | **Flag for review** — never auto-pass an uncertain ISI call | ISI presence is a mandatory, binary compliance gate; low confidence on a mandatory gate is not something to average away |
| ISI Classifier returns "absent" or "incomplete" | **Block publishing**, regardless of how confident every other stage was | Missing ISI is an automatic compliance failure independent of claim quality (chapter 00) |
| Content Comparator service unavailable/times out for an extracted claim | **Block** — an extracted claim with no comparator result is treated as unsupported, not as passed-by-default | Fail closed: absence of a "supported" result must never be silently upgraded to "supported" |
| Content Comparator's best match falls below the similarity threshold ("unsupported") | **Flag for review** | Could be genuinely new, not-yet-catalogued approved language — a human call, not an automatic block |
| Content Comparator's best match is found only in the historical/withdrawn index, not the active index (chapter 07) | **Flag for review with an explicit "supporting claim withdrawn/superseded" reason** | Must never be silently treated as supported just because *something* in the index matched |
| Write to the human-review queue fails after a flag decision | **Block and retry with alerting**, never drop the flag silently | A flagged item that never reaches the queue is worse than a slow pipeline — it can end up shipping unreviewed if the failure isn't surfaced |
| EventBridge-scheduled health check fails against a live Sagemaker endpoint | **Alert on-call; stop routing *new* content to that endpoint**, don't touch content already mid-flight | Contain a bad endpoint without turning a partial outage into a full one |
| Auto-scaling can't add capacity fast enough during a load spike | **Queue/backpressure at the API Gateway/Lambda layer with bounded retries**, not silent drops | A slow response is recoverable; a silently dropped request is not |

Two patterns worth naming explicitly if asked "how does this pipeline handle failure":

- **Every genuinely uncertain or failed model step fails closed, never open.** A missing, timed-out, or
  low-confidence result is never treated as equivalent to "checked and fine" — it either blocks
  publishing outright (ISI absence, extraction failure, comparator unavailability) or routes to the
  human review queue (low-confidence, low-similarity, or status-ambiguous results). Nothing in this
  table auto-passes content on the strength of a failure or a gap in the pipeline's own knowledge.
- **The review queue, not the model layer, is the actual safety net** — this is the same principle
  argued in chapter 03's threshold discussion and Q18/Q20 of the interview bank: because a false
  negative (an unsupported claim reaching publication) is categorically worse than a false positive (a
  reviewer spending two extra minutes dismissing a fine claim), the error-handling table above is
  deliberately asymmetric — it's much easier to end up in the review queue than to end up hard-blocked,
  and it's essentially impossible to end up auto-published on the back of an actual pipeline failure.

## The concurrency caveat: a model-artifact cache that outlives its own "new model deployed" signal

Chapter 05 describes each client's Sagemaker multi-model endpoint (`lilly-claims-mme`,
`astrazeneca-claims-mme`) loading models into memory on demand by `target_model` path, and chapter 04
describes the deployment step as "repoint the endpoint at a new registered version" rather than
retraining. Put those two together and a real caveat falls out: **the thing that decides *which* model
version a given request actually invokes is not the Sagemaker endpoint itself — it's whatever value the
Lambda routing layer has cached for "the current model version" at the moment it handles that request.**

A realistic implementation of the Lambda handler reads the current model version pointer (an SSM
Parameter Store value, or an environment variable) **once per execution environment**, caches it in a
module-level variable, and reuses it for every invocation that execution environment serves — which is
the efficient, standard pattern, since re-reading an SSM parameter on every single invocation adds
latency and cost for no benefit in the common case where the version hasn't changed. The problem is what
happens at the moment it *has* changed: AWS Lambda execution environments are reused across many
invocations for as long as they stay warm (this can be tens of minutes under steady traffic), and a warm
environment only re-reads that cached pointer on a cold start, not on a schedule. So immediately after a
retraining pipeline promotes a new model version and fires a "new model deployed" signal, **any
already-warm Lambda execution environment keeps routing requests to the old `target_model` path until
that specific environment happens to recycle** — which is not immediate, not synchronized across the
fleet, and not something the deployment step controls directly.

Concretely: two content items submitted seconds apart, load-balanced across two different warm Lambda
execution environments, can be classified by two different model versions — one already picked up the
new pointer, the other is still serving off a cached pointer from before the deploy — with **no record
anywhere of which version actually produced which result**, unless the classification result itself is
stamped with the model version that generated it. That last point is exactly what
`notebooks/07_model_version_tag_propagation_demo.ipynb` demonstrates: making every classification result
carry its producing model version as a first-class field turns an invisible mixed-version window into a
directly detectable one — you can query "how many of the last hour's results came from the outgoing
version" instead of having no way to even ask the question.

**Two realistic ways to harden this, worth naming as a pair:**

1. **Bound the staleness window explicitly**, by giving the cached pointer a short TTL (say, re-check
   the SSM parameter every 2–5 minutes even on a warm environment) rather than relying on however long
   that Lambda environment happens to stay alive. Cheap, and turns an unbounded, traffic-dependent
   staleness window into a small, known one.
2. **Stamp every result with the model version actually used for that specific invocation**, read
   directly off the `target_model` value the call resolved to, not the value the deployment step
   *intended* to be live. This doesn't shrink the staleness window at all, but it makes the window
   observable and auditable — which, for a compliance pipeline, is arguably the more important property:
   knowing precisely which nine requests out of the last thousand were classified by the outgoing model
   version is a very different operational position than not knowing a mixed-version window happened at
   all.

The honest framing: *"this wasn't caught until a retraining cycle actually shipped, because the failure
mode is a handful of requests briefly using the previous model version, not an outage — the fix that
matters most isn't eliminating the window, which is expensive to do perfectly against Lambda's own
execution-environment lifecycle, it's making the window visible so it can be quantified and bounded
rather than silently assumed away."*

## Four bugs found and fixed — realistic, source-typical narratives for this kind of pipeline

Written in the same spirit as a "tell me about a bug you found" story — plausible, specific, and paired
with what practice would have caught each one earlier, not vague generalities.

**1. A tokenizer-version mismatch between training and serving silently degraded accuracy.** The
fine-tuning job (chapter 02) loaded its tokenizer via `AutoTokenizer.from_pretrained("bert-base-uncased")`
with no pinned revision hash, and so did the serving container — except the serving container's image
was rebuilt weeks later, picked up a newer default snapshot of the same named checkpoint from the model
hub, and that snapshot's tokenizer config handled a handful of special characters (en-dashes and the
"±" symbol common in efficacy figures like "42% ± 3%") differently than the version training had used.
Nothing crashed — the model just ran on slightly different token sequences than it was trained on for
any claim containing those characters, quietly shaving accuracy specifically on numeric-efficacy claims
without any error to point at. *What would have caught this earlier:* pinning an exact revision hash (not
just a model name) for the tokenizer in both the training script and the serving container's
requirements, and a CI check that hashes the tokenizer config actually packaged inside the model
artifact and fails the deploy if the serving container's tokenizer config doesn't match it byte-for-byte.

**2. A class-label off-by-one after a schema migration silently mislabeled a whole batch.** When a new
claim-type tag was added (splitting a combined "dosing-comparative" tag into separate "dosing" and
"comparative" tags), the training config's ordered label list was updated in one place, but the Lambda
routing layer's response-decoding step kept its own **separately maintained, hardcoded copy** of the
ordered tag list to turn the model's sigmoid output positions back into human-readable tag names.
Inserting the new tag in the middle of the training config's list shifted every subsequent index by one
in the model's actual output, but the serving-side hardcoded list wasn't updated to match — so every
claim that should have been tagged, say, "safety" for weeks came back labeled "comparative" instead,
silently, for the entire batch of content processed until the mismatch was noticed. *What would have
caught this earlier:* never maintain the label-to-index mapping in two independently-edited places —
store the ordered `label_names` list as metadata inside the model artifact itself and have serving read
it directly from there for decoding, and add a golden-set regression test asserting *label names*
(not raw indices) on a fixed set of known inputs, run automatically on every artifact promotion.

**3. A similarity threshold left at its notebook-experiment default flooded reviewers with false-positive
duplicate flags.** The Content Comparator's TF-IDF match threshold was tuned during offline experimentation
against a small labeled set (the same shape of value as `TFIDF_FLAG_THRESHOLD = 0.35` in this course's own
`03_similarity_content_comparator.ipynb`), and that literal value shipped unchanged as the production
default. At notebook scale, against a library of a few dozen claims, 0.35 was a reasonable cutoff; at
production scale, against a library of thousands of approved claims spanning many therapeutic areas, the
same threshold cleared far more easily on shared generic pharma vocabulary alone ("clinical," "patients,"
"significant," "compared to placebo") even between claims about entirely unrelated drugs — producing a
sustained flood of false-positive "supported/near-duplicate" flags that buried reviewers in noise and
undermined trust in the tool's flags generally. *What would have caught this earlier:* treating similarity
thresholds as environment-specific configuration re-validated against production-scale precision/recall
curves before go-live, not a constant carried over unchanged from a small experimental notebook — a
"threshold audit against realistic data volume" line item in a pre-launch checklist would have caught
this before reviewers did.

**4. A retraining job evaluated against a stale cached copy of the approved-claims library and had to be
rerun.** This is the direct, concrete version of chapter 07 Part 5's forward-looking concern actually
happening: the retraining pipeline's data-validation stage (chapter 04, stage 1) pulled its approved-library
reference snapshot from an S3 path refreshed on its own independent weekly schedule, decoupled from the
actual cadence at which legal revised claims. A retraining run landed in the same week legal had withdrawn
several claims after a labeling update, but the snapshot job hadn't run since before that revision — so the
newly retrained comparator's embedding index was built from, and evaluated against, wording that had
already been retired. It passed its evaluation gate cleanly (nothing about the gate's accuracy comparison
would have flagged this — the reference set was internally consistent, just stale) and was only caught
when a reviewer noticed the newly deployed comparator matching new content against claim language legal
said had been pulled months earlier. The job had to be manually rerun after forcing a fresh library
snapshot. *What would have caught this earlier:* exactly the fix chapter 07 proposes — the data-validation
stage explicitly asserting the reference snapshot's timestamp is at least as recent as the approved
library's own last-revision high-water mark, and failing the pipeline run outright rather than silently
training and evaluating against ground truth that's already out of date.

The common thread across all four, worth stating as the takeaway: **none of the four produced an error or
a crash** — a tokenizer silently tokenizes differently, an off-by-one silently relabels, a threshold
silently over-triggers, a stale snapshot silently trains cleanly against outdated ground truth. All four
are the textbook "the pipeline ran without throwing, so it looked healthy" class of bug that only
targeted regression tests (a golden-set with known expected labels, an artifact/config hash check, a
threshold re-validated at realistic scale, a data-freshness assertion) catch — generic uptime/error-rate
monitoring would have missed every single one of them, because nothing about them looked like a failure
from the outside.

## Concrete thresholds, timeouts, and batch sizes — and why each value is what it is

- **Sagemaker `invoke_endpoint` timeout: 60 seconds**, set explicitly on every call from the Lambda
  routing layer, with the Lambda function's own timeout budget set to 90 seconds — enough headroom for
  the invocation call plus response handling without the Lambda timing out *underneath* a Sagemaker call
  that's still legitimately in flight, which would otherwise turn a slow-but-completing inference into an
  apparent failure.
- **Batch size for claim extraction on long documents: up to 50 sentences per `invoke_endpoint` call**,
  chosen to stay comfortably under Sagemaker real-time endpoints' request payload ceiling while still
  amortizing the fixed per-call overhead across a meaningful chunk of a document rather than invoking the
  endpoint sentence-by-sentence for a 40-page leave-behind.
- **Retry policy on Sagemaker call failures: 3 attempts with exponential backoff and jitter** (roughly
  1s, 2s, 4s base delays), tuned so the retry window is short enough not to visibly stall a
  human-in-the-loop review session, but long enough to ride out a transient throttle or a brief cold-load
  delay on a multi-model endpoint's less-recently-used model.
- **EventBridge health-check interval: every 5 minutes per client endpoint** — frequent enough to catch a
  degraded endpoint well before it accumulates a meaningful amount of real failed traffic, infrequent
  enough not to itself become a meaningful fraction of the endpoint's invocation volume.
- **TF-IDF flag threshold: re-tuned per environment, not the notebook's 0.35 default** — the concrete
  number that matters less than the practice bug #3 above establishes: whatever the production value is,
  it's derived from a precision/recall curve against production-scale data, revisited as the approved
  library grows, not inherited unchanged from an experimentation notebook.

## The hardcoded internal service token: a real-shaped hardening gap

A plausible, specific hardening gap worth naming candidly rather than glossing over: the static bearer
token the ECS Fargate orchestration layer uses to authenticate its calls into the Lambda routing layer —
proving "this call originated from our own API tier, not an external caller" — is baked into both tiers'
deployment configuration as a **literal environment-variable value set at build time**, rather than a
Secrets Manager reference resolved at invocation time the way the Lambda layer's *other* credentials
already are (chapter 05). Concretely, this means the token's value is visible in plaintext to anyone with
read access to the ECS task definition or the Lambda function's configuration in the AWS console or CLI,
and rotating it requires a coordinated redeploy of both tiers simultaneously rather than a single
Secrets Manager rotation event that both sides pick up transparently. If that value ever leaked — a
misconfigured log line, a screenshot in a support ticket, an engineer with broader read access than they
needed — anyone holding it could call the Lambda routing layer directly, bypassing the ECS layer's own
auth and client-scoping logic entirely. The fix is mechanical and low-risk: move the token into Secrets
Manager, have both tiers fetch it at invocation/cold-start time instead of reading a baked-in environment
variable, and rotate it on a schedule — the same category of fix, and the same reason it's worth naming
candidly instead of hiding it, as the hardcoded `SessionMiddleware` secret in the Capco Document Uploader
course.

## Tying it back

Production-grade for a compliance pipeline isn't "the models are accurate" — it's "every failure mode is
known, fails in the safe direction, and is bounded and observable rather than silently absorbed." This
pipeline's real profile, stated plainly: a genuinely asymmetric, fail-closed error-handling table that
never lets an uncertain or failed step masquerade as a clean pass, one well-understood scaling caveat in
how model-version routing interacts with Lambda's own execution-environment lifecycle (with a concrete,
proportionate fix already identified — bound the staleness window and make it observable), four realistic
bugs that are all in the same "ran without error, looked healthy, was quietly wrong" family — exactly the
class of bug that targeted regression tests catch and generic monitoring doesn't — and one honestly-named
hardening gap that's a small, mechanical fix away from closed.
