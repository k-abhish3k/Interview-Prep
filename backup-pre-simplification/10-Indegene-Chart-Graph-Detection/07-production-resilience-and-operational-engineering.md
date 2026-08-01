# 07 — Production Resilience and Operational Engineering

This is the "production grade, not just architecture diagrams" chapter for the chart/graph detection
pipeline — mirroring course 05's chapter 06 for this project's stack: a realistic error-handling
table for a serverless CV inference path, the genuine serverless-concurrency scaling caveat that comes
with sharing one Lambda function across two paying clients, four concrete "bug found and fixed"
narratives specific to an object-detection deployment, the concrete threshold/memory/timeout values
worth being able to defend, and one candidly-named hardening gap.

**The same honesty note from Chapter 06 applies throughout this chapter:** there is no source
repository backing this course to confirm any of the below against. Every error-handling behavior, bug
narrative, and parameter value here is an **illustrative, technically grounded reconstruction** of what
a Lambda + API Gateway + Sagemaker-backed YOLOv5 deployment like the one in Chapter 04 would plausibly
look like in production, and what a candid, experienced engineer would say if asked to walk through it
— not a verified account of a specific incident log. Present it that way if asked directly.

## The realistic error-handling table

The central judgment call for any request-response inference API is the same one course 05 makes for
its upload service: not every failure deserves the same response. A malformed input, a transient
infrastructure hiccup, and a genuine model failure are three different problems and should be handled
three different ways — retry, fail fast, or degrade gracefully, matched to the actual failure mode
rather than one blanket behavior for everything.

| Failure | Classification | What should happen |
|---|---|---|
| Malformed input image (corrupt bytes, unsupported format, unreadable by PIL) | Client data problem — permanent | **Fail fast.** Structured `400`, no retry — the same bytes will never decode successfully, so retrying only delays the caller's inevitable error and burns an invocation |
| Oversized image (exceeds a defined max dimension or payload size) | Client data problem — permanent | **Fail fast.** Structured `413`/`400` *before* attempting to decode or run inference — attempting anyway is exactly how the memory bug below happens |
| Cold-start initialization exceeding the configured Lambda timeout | Transient infrastructure problem | **Retry.** The same request, retried, likely lands on an already-warm container (or one that now has enough time budget) and succeeds — this is not a data problem, and failing it back to the caller as if it were one is misleading |
| Lambda/API Gateway throttling (`429` / `TooManyRequestsException`) under burst load | Transient capacity problem | **Retry with exponential backoff.** Capacity, not data, is the constraint; a well-behaved caller (the ECS orchestration layer) should back off and retry, not fail the document outright |
| Genuine detection failure — the model raises mid-inference (e.g., an edge-case image producing an unexpected tensor shape) | Real bug / model-input mismatch | **Fail fast for that image, degrade gracefully for the batch.** Don't blindly retry — the same input will fail the same way again — but log and flag the one image and keep processing the rest of the batch/document set rather than aborting the whole run over one bad page |
| Downstream S3 write failure for detection results | Transient infrastructure problem | **Retry a bounded number of times, then dead-letter.** A completed detection should never be silently dropped — park it for reprocessing rather than losing it |
| Sagemaker endpoint scaling/cold event (if inference is routed to a Sagemaker real-time endpoint rather than run in-Lambda) | Transient infrastructure problem | **Retry with backoff**, same reasoning as the Lambda cold-start row |

Two patterns worth naming explicitly if asked "how does this pipeline handle failure":

- **Data-shaped failures fail fast and never retry blindly.** Retrying a malformed image or an image
  that trips a genuine model bug doesn't fix anything — it just wastes Lambda invocations, adds
  latency, and delays a caller's eventually-inevitable error. The signal to build toward is: if the
  same input, retried, cannot succeed, don't retry it.
- **Infra-shaped failures should retry, ideally with backoff**, because the same request very likely
  succeeds once the transient condition (a cold start finishing, a throttle window passing, a brief S3
  blip) clears. The failure mode worth actively guarding against is inverting these two — retrying a
  genuine detection failure as if it were transient (burning invocations for nothing), or treating a
  cold-start timeout as a real detection failure and giving up on a perfectly valid image.

## The concurrency caveat: one shared Lambda function, two clients, no fairness by default

This is this project's analog to course 05's "background loop is per-process, not a singleton"
caveat — a real, bounded scaling risk that comes from a technically reasonable design decision (one
shared Lambda function and code path serving both Eli Lilly and AstraZeneca, per Chapter 04 and
`00-README.md`'s multi-client isolation model) once real concurrent production load actually exercises
it.

**Lambda concurrency is a shared, account-level resource pool by default.** If the `/detect-charts`
Lambda function has no reserved concurrency configured, every invocation — regardless of which client
it serves — draws from the same account-wide concurrency ceiling. Concretely: if Eli Lilly's
document-ingestion process kicks off an end-of-day batch that fires several hundred concurrent
detection requests, and AstraZeneca's smaller, latency-sensitive real-time requests arrive in that same
window, AstraZeneca's requests can get throttled (`429`) or queued — **not because AstraZeneca's own
traffic changed at all**, but because a shared resource pool has no fairness guarantee across callers
by default. This is never a data-corruption risk — Lambda's own throttling behavior returns a clean,
retryable error rather than silently dropping or corrupting a request — but it's a real,
service-quality-degrading risk: one client's burst pattern determines the other client's latency, which
is exactly the kind of cross-tenant coupling the isolation design elsewhere in this pipeline (per-client
S3 prefixes, scoped IAM) is built to avoid everywhere else.

**Concrete fixes, worth naming as a set rather than one silver bullet:**

1. **Reserved concurrency per client**, via separate Lambda aliases/versions — e.g., a `batch` alias
   handling large end-of-day submissions capped at its own reserved ceiling (illustratively, ~150 out
   of an account pool), and a `realtime` alias for latency-sensitive interactive calls with its own
   separate reservation, so a burst on one alias physically cannot consume the concurrency the other
   alias needs.
2. **Provisioned Concurrency on the realtime alias**, sized to each client's typical baseline load (a
   handful of warm environments, not peak capacity), so common-case interactive requests avoid cold
   starts regardless of what the batch alias is doing.
3. **Per-client throttling at API Gateway**, via separate usage plans/API keys with their own rate and
   burst limits (illustratively, on the order of tens of requests per second steady-state with a modest
   burst allowance per client) — enforcing fairness before a spike even reaches Lambda's concurrency
   pool, rather than relying on Lambda-level reservation alone.

The honest framing, matching course 05's tone on its own scaling caveat: *"this risk is real and worth
naming, but it's bounded and well-understood — the worst case is a throttled, retryable request for the
other client during a burst window, never lost or corrupted data, and the fix (per-client reserved
concurrency plus API Gateway usage plans) is a configuration change, not a redesign."*

## Four bugs found and fixed — object-detection-deployment specific

Genuine, illustrative bug narratives shaped around this project's actual stack (a fine-tuned YOLOv5
model behind a Lambda inference handler), the kind of concrete material a "tell me about a bug you
found" question is fishing for:

**1. Confidence threshold left at a validation-sweep value instead of a production value.** YOLOv5's
own evaluation tooling (`val.py`) sweeps confidence down to a near-zero value (illustratively, `0.001`)
so it can trace out a full precision-recall curve across every possible operating point — that's the
correct value *for computing a PR curve*, and the wrong value *for a production endpoint returning
boxes to a caller*. An early version of the Lambda handler's `confidence_threshold` default carried that
validation-sweep value forward instead of the production-appropriate `0.25` (the value
`notebooks/04_lambda_inference_handler_demo.ipynb`'s handler actually defaults to), so early responses
came back with dozens of near-zero-confidence boxes per image — technically "detections," but noise
that buried the handful of genuinely high-confidence chart regions the downstream pipeline actually
needed. *What would have caught it earlier:* a smoke test asserting `num_detections` on a known
reference image stays within an expected small range **at the actual deployed default**, not just a
unit test of the model's raw output in isolation — the bug was in the handler's default parameter, not
the model.

**2. Lambda timeout left at the platform default of 3 seconds.** AWS Lambda's default timeout for any
newly created function, if not explicitly configured, is 3 seconds. Loading `weights.pt` plus
initializing the PyTorch runtime alone — the cold-start cost Chapter 04 describes — routinely takes
several seconds on its own, before any inference even starts. The practical result: every cold start
(the first invocation of a fresh container, and any invocation after Lambda recycles an idle one) timed
out before the handler could finish loading the model, while every warm invocation succeeded instantly
— a classic "works every time I manually test it a second time in a row, fails invisibly on the very
first request of a burst" bug, since a manual test naturally re-invokes a now-warm container. *What
would have caught it earlier:* a CloudWatch alarm on Lambda `Duration` approaching `Timeout` combined
with `Throttles`/`Errors`, and — more directly — a synthetic canary that deliberately invokes the
endpoint after an idle period long enough to force a cold start, rather than a test suite that only ever
hits an already-warm function.

**3. A retrained model deployed without any version identifier changing anywhere.** A new fine-tune —
improved recall on forest plots, say — got pushed into the deployed container/model artifact, but
nothing in the response payload or deployment metadata recorded that a change had happened at all; the
only version-adjacent field was `model_loaded_at`, a cold-start timestamp that changes on every
container recycle **regardless of whether the underlying weights changed**. When the monthly per-client
detection-count report (Chapter 06) showed a sudden jump the following cycle, there was no way to
attribute it to "the client sent more documents" versus "the model changed and is now more sensitive" —
exactly the model-version-attribution gap Chapter 06 is built around. *What would have caught it
earlier:* a deployment checklist or CI gate that **blocks the release** unless an explicit
`MODEL_VERSION` tag changes whenever the model artifact's hash changes — making the version bump a
build requirement, not a courtesy someone might forget under deadline pressure.

**4. Image preprocessing applied inconsistently between training and serving.** YOLOv5's own training
pipeline uses **letterbox resizing** — scale the image to fit within the target size (640×640) while
preserving aspect ratio, then pad the remainder with neutral gray — because every bounding box the
model was trained on is labeled relative to that letterboxed geometry (Chapter 02). An early version of
the Lambda handler instead used a naive stretch-resize (`image.resize((640, 640))`-shaped, ignoring
aspect ratio) — easy to write without pulling in YOLOv5's own preprocessing utility, and it doesn't
throw an error or fail any obvious check. It just quietly distorts every non-square input image's
aspect ratio in a way the model never saw during training, producing systematically shifted and
undersized boxes on any landscape-oriented slide (the common case for this project) — a silent accuracy
regression invisible in a spot check against a handful of conveniently-square test images. *What would
have caught it earlier:* a preprocessing-parity test that runs the same input image through both the
training-side letterbox function and the serving-side handler's preprocessing and asserts the resulting
arrays match pixel-for-pixel, run in CI — plus periodically re-running the IOU/TPR evaluation
(Chapter 03) against the **actual deployed serving path's** preprocessing, not only the training
pipeline's own validation split, which would never surface a train/serve skew bug because it never
touches the serving code at all.

The common thread across all four, worth stating as the takeaway: **each one is a "the code runs
without error, so it looks fine" bug** — a noisy-but-non-crashing threshold, a timeout that only bites
on the very first request of a burst, a missing tag nobody notices until a report looks wrong a month
later, and a resize that silently degrades accuracy rather than throwing. None of the four would be
caught by testing the model in isolation on its own validation set; all four needed a test (or a
monitoring signal) that exercised the actual deployed serving path, defaults included.

## Concrete parameter values, and why they matter

| Setting | Illustrative value | Why |
|---|---|---|
| Lambda memory | ~3,008 MB | Sized to comfortably hold the loaded YOLOv5 weights, PyTorch runtime overhead, and a decoded high-resolution page image simultaneously — found empirically by watching CloudWatch memory-utilization metrics under realistic (not toy) input sizes and increasing until utilization plateaus with headroom; too low silently truncates or OOM-kills on large scanned documents rather than failing with a clear error |
| Lambda timeout | ~60 seconds | Covers worst-case cold-start-plus-inference latency for the largest expected input image, with margin — set against the *actual* cold-start cost (bug #2 above shows what happens when this is left at Lambda's 3-second platform default instead of being explicitly configured) |
| Inference confidence threshold | 0.25 (production) vs. 0.001 (validation-sweep) | The production value is a deliberate recall-leaning business choice (Chapter 03); the validation-sweep value exists purely to trace a full precision-recall curve during evaluation and should never reach a production response payload (bug #1 above) |
| NMS IOU threshold (post-processing) | 0.45 | Controls how aggressively overlapping candidate boxes around the *same* chart get merged during Non-Maximum Suppression — a completely different IOU threshold from the strict 0.85 **evaluation** threshold in Chapter 03; conflating "the IOU used to merge duplicate boxes" with "the IOU used to judge whether a detection counts as correct" is a mistake worth actively avoiding out loud in an interview, per Chapter 02's own note on this |
| Reserved concurrency per client | ~150 (batch alias) | Caps one client's burst so it cannot consume the concurrency the other client's traffic also needs (see the concurrency caveat above) |
| Provisioned Concurrency per client (realtime alias) | 2–5 warm environments | Sized to typical baseline latency-sensitive load, not peak — deliberately eating occasional burst-time cold starts rather than paying for 24/7 peak capacity |
| API Gateway per-client throttling | tens of req/s steady-state, modest burst allowance | Enforces fairness between clients before a spike ever reaches Lambda's concurrency pool |

## One candidly-named hardening gap

Worth naming plainly rather than glossing over: **the fine-tuned model artifact's location is baked
into the container image as a literal `COPY weights.pt` path (per Chapter 04's Dockerfile sketch)
rather than read from an environment variable pointing at an S3 model-artifact location, and there is
no `MODEL_VERSION` value stamped anywhere in the image or the response payload.** This is the same
underlying gap Chapter 06 is built around, and it has two concrete, negative consequences worth
naming together:

- **Every model retrain requires a full container rebuild and redeploy**, rather than a lightweight
  configuration change (swap an environment variable, point at a new S3 key) — slower, riskier, and
  more coupled to the deployment pipeline than it needs to be.
- **There is no way to answer "which model produced this detection" after the fact**, which is exactly
  bug #3 above and exactly the problem Chapter 06's `model_version` field is designed to solve.

The fix is low-risk and mechanical, in the same spirit as course 05's hardcoded-secret fix: read the
model artifact's S3 location and a `MODEL_VERSION` tag from environment variables (injected via Lambda
configuration, not baked into the image), load the artifact from S3 at cold-start init instead of
`COPY`-ing it into the image, and stamp `MODEL_VERSION` onto every response payload. None of this
changes the inference logic itself — it's a packaging and observability fix, not a model change.

## Two things worth highlighting positively

Not everything here is a gap — two existing patterns (per Chapter 04 and `notebooks/04_lambda_inference_handler_demo.ipynb`) are genuinely good practice, worth citing if asked "what did you do right":

- **The model is loaded once at module scope, outside the handler function.** This is the correct
  cold-start mitigation (Chapter 04) and it's already the shape of the reference handler — the
  (simulated) expensive load cost is paid once per warm container, never repeated per-invocation.
- **Defensive image decoding with an explicit `.convert("RGB")` step.** Guarding against palette-mode
  or CMYK-encoded input images (a real, if unglamorous, class of "malformed input" that a document
  pipeline ingesting scanned and exported content will genuinely encounter) rather than assuming every
  incoming image already arrives as clean RGB — a small, specific detail that prevents a real class of
  decode-time failure before it ever reaches the model.

## Tying It Back

Production-grade for a serverless CV inference path means the same thing course 05's chapter 06 argues
for a web service: knowing precisely which failures should retry, which should fail fast, and which
should degrade gracefully rather than picking one behavior for everything; naming the real scaling
caveat that comes from sharing capacity across tenants (here, Lambda concurrency across two clients,
not a per-process background loop) with a concrete, bounded worst case and an already-identified fix;
being able to walk through real bugs with specifics — a threshold, a timeout default, a missing tag, a
resize mismatch — and exactly what would have caught each one earlier; and naming one hardening gap
candidly, with the fix already in mind, rather than either overclaiming polish or apologizing for it.
