# 06 — Detection Deduplication and Model-Version Drift

## Why this chapter exists

Course 03 has a chapter built around a real interview question the candidate was actually asked:
"for revised versions of the same document, how are you handling those?" That question generalizes
cleanly to any production ML pipeline that (a) processes documents that get resubmitted or corrected
over time, and (b) retrains and redeploys its model periodically — which this project does on both
counts. An interviewer who hears "fine-tuned YOLOv5, deployed on Lambda, Sagemaker-hosted, running in
production for two pharma clients" has an obvious, sharp follow-up available: *what happens when the
same document comes through twice, and what happens to old detection results once you retrain the
model?* This chapter is that answer, built out with the same rigor as course 03's chapter 05.

**A note on how this chapter should be read, unlike course 03's:** course 03 was rebuilt from real
source code, so its versioning chapter could point at an actual function name and say "this exact
logic doesn't exist." There is no equivalent source repository backing this course — everything
below, including "what a realistic implementation does today," is an **illustrative, technically
grounded reconstruction** of how a Lambda + API Gateway + Sagemaker chart-detection pipeline like the
one described in Chapter 04 would plausibly behave, and how a thoughtful engineer would reason about
closing the gap — not a confirmed account of implemented code. Treat every claim of the form "this is
what happens today" as "this is what would realistically happen given the architecture described in
Chapter 04 and `00-README.md`," and say so explicitly if asked in an interview.

The honest shape of the answer has five parts: what a realistic implementation does today (nothing
prevents duplicate detection runs, and nothing tags a detection with the model that produced it), a
manual workaround using AWS primitives that already exist, two easily-conflated concerns worth naming
separately, a genuine proposed design to close the gap, and the forward-looking tie to a downstream
analytics layer that shows why this matters beyond bookkeeping.

## Part 1 — What a realistic implementation does today: no dedup, no version stamp

Chapter 04 and `00-README.md` describe the real-time inference path plainly: a document lands in a
client's S3 prefix, the ECS Fargate orchestration layer routes it to the Lambda + API Gateway
detection endpoint, the Sagemaker-hosted fine-tuned YOLOv5 model runs inference, and results are
written back to S3 for the downstream pipeline stage. Nothing in that description — and nothing a
realistic build of it would plausibly add without deliberate effort — asks either of these two
questions before paying for a Sagemaker/Lambda invocation:

1. **"Have I already produced detections for this exact image content?"** No content hash, checksum,
   or byte-for-byte comparison happens before the detection call. `notebooks/04_lambda_inference_handler_demo.ipynb`'s
   handler is representative of the real shape here: it decodes whatever bytes arrive in the request
   body and runs inference unconditionally — there's no lookup step before that decode call asking
   whether this content has been seen before.
2. **"Which model version produced this result?"** The handler's response payload (`image_size`,
   `num_detections`, `detections`, `inference_ms`, `model_loaded_at`) has no field naming the model
   artifact or fine-tune iteration that generated the boxes. `model_loaded_at` is a wall-clock
   cold-start timestamp — it changes every time a fresh Lambda execution environment spins up,
   **regardless of whether the underlying `weights.pt` changed at all** — so it cannot be used, even
   accidentally, as a stand-in for model identity.

The practical consequence: submitting the exact same document bytes twice — because an upstream
retry fired twice, because a client's ingestion pipeline re-exported a whole deck after fixing a typo
on one slide, or because someone manually resubmitted a document they weren't sure had gone through —
produces **two entirely independent sets of bounding-box detections**, written to two S3 locations,
with no relationship recorded between them. And retraining the model and swapping the deployed
artifact produces detections with no tag distinguishing "these boxes came from the model that was
live in March" from "these boxes came from the model that replaced it in June."

This is the honest first sentence of the interview answer: *"There's no content-based deduplication
before the inference call, and no model-version field stamped on a detection result — every
invocation is treated as independent, and nothing at the detection layer distinguishes a resubmitted
document from a brand-new one, or a detection produced by last quarter's model from one produced by
this quarter's."*

## Part 2 — The manual workaround, using pieces that already exist in AWS

Nothing about this gap is unsolvable with what's already sitting in the AWS account — it's just not
wired together for this purpose, exactly the "the pieces exist, nothing automates them" shape course
03's chapter 05 describes for its own versioning gap:

1. **S3 already computes a content checksum on every object.** Every S3 object has an `ETag` (an
   MD5-based content hash for non-multipart uploads, or a proper SHA-256/CRC32C checksum if S3's
   additional-checksum feature is enabled on the bucket). An engineer investigating a suspected
   duplicate can pull the `ETag`/checksum for two object keys and compare them directly — no code
   change required, just an `aws s3api head-object` call on each key.
2. **CloudWatch Logs already captures every Lambda invocation.** With a small addition to the handler
   (logging the incoming image's hash and the S3 key alongside each invocation — not present in the
   handler as described in Chapter 04, but a one-line addition), a **CloudWatch Logs Insights** query
   can retroactively answer "has this content hash been submitted before" by scanning invocation logs
   for a matching hash. Without that log line already in place, this workaround is closer to
   forensic reconstruction than a live check — worth being honest about.
3. **Manual cross-referencing and deletion.** Having identified two detection outputs for the same
   logical document (by checksum match or by a human recognizing the same document title/filename
   pattern), an engineer can manually decide which S3 result object is authoritative and delete or
   move the stale one out of whatever downstream store the reporting layer reads from.

The precise, defensible statement to make about this: *"the primitives to build a real dedup check
already exist in the platform — S3 checksums, CloudWatch Logs Insights — the gap isn't that AWS lacks
the tools, it's that this pipeline doesn't yet call them for this purpose before invoking the model."*

**Caveat worth stating alongside this, matching course 03's:** this manual process, even done
carefully, doesn't recover the *lineage* between the two runs — deleting the stale detection set loses
the fact that it was ever produced, so anything that referenced its `detection_run` (an audit log, a
downstream report that already counted it) doesn't get corrected, only silenced going forward. That's
exactly the gap Part 4 is designed to close.

## Part 3 — Two concerns that are easy to conflate, and shouldn't be

It's tempting to reach for one fix ("just dedupe on the document") and assume it covers both problems
this chapter is about. It doesn't, and treating them as one concern is a mistake worth actively
avoiding in an interview answer:

- **Concern A — "this is a corrected/resubmitted version of a document already processed."** This is
  a question about the *document's* identity over time: did the same logical document (same client,
  same underlying content, possibly touched up) come through the pipeline more than once? The fix for
  this is a **deduplication and supersession** problem — detect the repeat, and either short-circuit
  (true duplicate, same bytes) or link the new run to the old one as a replacement (corrected
  resubmission, different bytes, same logical document) rather than letting both sit as independent,
  unrelated results.
- **Concern B — "the model that produced this result is not the same model that produced that other
  result."** This is a question about *model* identity over time, completely orthogonal to whether the
  document changed. Two runs on two genuinely different, never-before-seen documents can still be
  incomparable if one ran against the March fine-tune and the other against the June fine-tune — the
  fix here isn't deduplication at all, it's **attribution**: every detection result needs to carry
  which model produced it, so a difference in output can be correctly traced to "the model changed"
  rather than misdiagnosed as "the document population changed" (or vice versa).

The place these two concerns most dangerously get conflated: a reprocessed document shows a
*different* set of detections than the first time it went through the pipeline. That difference has
**three genuinely different explanations**, and jumping to the wrong one is exactly the kind of error
this chapter exists to prevent naming clearly:

1. **Same document, same model, processed twice** — a pure duplication bug. The two detection sets
   should be identical (deterministic inference); if they're not, something else is wrong (e.g.
   non-determinism in preprocessing), but the *presence* of two sets at all is the bug to fix,
   regardless of whether their contents match.
2. **Corrected document, same model** — a legitimate change in *input*, not a bug. The new detections
   are correct and expected to differ; what's needed is bookkeeping (a supersession link), not
   suppression.
3. **Same or different document, different model version** — an expected, and often *desirable*,
   change in the model's behavior (a retrain that improved recall on forest plots, say). This is not
   something to "fix" by deduping it away — it's something to make *attributable*, so a stakeholder
   asking "why did detection counts change" gets a correct answer instead of a guess.

The precise way to draw this boundary out loud: *"deduplication answers 'has this content been seen
before'; model-version attribution answers 'which model produced this result' — those are two
different axes, and a system that only solves one of them still leaves the other ambiguous. I'd be
careful not to present a content-hash dedup check as if it also solved the model-attribution problem,
because it doesn't touch it at all."*

## Part 4 — A genuine proposed design for both

This is explicitly a **proposal** — an illustrative, not-implemented design, labeled as such
throughout, in the same spirit as course 03's `document_group_id`/`supersedes_document_id` proposal.

**A `detection_runs` record, stamped on every inference call**, owned by the ECS Fargate orchestration
layer (the natural place for it, per Chapter 04's argument that cross-cutting, stateful concerns like
multi-client routing belong in the orchestration layer rather than duplicated into every Lambda
invocation):

```python
# Proposed additions -- not implemented today, illustrative only
detection_run_id   = uuid4()
document_id        = <client-provided or orchestration-assigned logical document identifier>
content_hash       = sha256(raw_image_bytes)      # dedup key: identical bytes -> identical hash
model_version      = "yolov5-chart-finetune-2026-05-14"  # the deployed model artifact's identity,
                                                          # NOT a cold-start timestamp
supersedes_run_id  = <detection_run_id of the prior run for this document_id, if any, else null>
client             = "eli-lilly" | "astrazeneca"
created_at         = timestamp
```

**Before invoking the Sagemaker/Lambda endpoint**, the orchestration layer would look up
`content_hash` against prior runs:

- **Hash found, `model_version` unchanged** → true duplicate under the same model. Short-circuit:
  return the cached prior detections without paying for another inference call. This is the case
  Part 1 currently handles by re-running inference every time — a genuine cost saving as well as a
  correctness fix.
- **Hash found, `model_version` different** → the model was retrained since this exact content was
  last processed. Re-run inference (a legitimate "reprocess under the new model" case), but write the
  new row with `supersedes_run_id` pointing at the prior run for the same hash — both remain queryable,
  and neither silently overwrites or duplicates the other in a downstream store.
- **Hash not found, but `document_id` matches a document that has been processed before** → this is
  Concern A, a corrected resubmission. Re-run inference (the content genuinely changed, so it must be
  reprocessed) and set `supersedes_run_id` to the latest confirmed run for that `document_id`, marking
  the prior run as superseded rather than leaving both live in whatever the downstream store considers
  "current."
- **Neither matches** → a genuinely new document. Run inference normally, with no `supersedes_run_id`.

Every detection result, regardless of which of the four paths it took, gets `model_version` stamped
on it directly — replacing the current, misleading `model_loaded_at` field with an actual, stable
model identity that survives container recycling and cold starts untouched.

**Where this genuinely can't be a one-layer fix, worth naming explicitly:** the Lambda inference
handler itself is, and should stay, stateless and fast (Chapter 04's whole argument for serverless
here rests on that). The `content_hash`/`document_id` lookup has to live in a layer that already holds
state across invocations — which is exactly why this proposal puts it in the ECS Fargate orchestration
layer rather than trying to bolt a lookup table onto the Lambda function itself. A DynamoDB table
(single-digit-millisecond lookups, a natural fit for a hot-path pre-check keyed by hash) is a more
realistic backing store for this than a SQL table here, given the orchestration layer's AWS-native
context — a genuine, defensible implementation detail, not an arbitrary choice.

## Part 5 — The forward-looking tie to downstream reporting: why this is more than bookkeeping

`00-README.md`'s pipeline diagram ends with "detection results written back to S3... for the
downstream pipeline stage to pick up." The most natural next stage for that data, for a production
pharma-content pipeline serving two paying clients, is some form of **per-client analytics or
reporting layer** — a monthly "charts/graphs detected and routed" count feeding a client status
report, a dashboard tracking detection volume trends over time, or an MLR-review queue sized off how
many chart regions were flagged this cycle. This is exactly the kind of downstream layer this course's
`00-README.md` and Chapter 04 gesture at without fully specifying — plausible, not confirmed, in the
same way course 03's chapter 05 treats its own downstream retrieval layer.

If that reporting layer exists and neither Part 4 fix is in place:

- **No dedup/supersession** → a corrected resubmission of a document silently produces a *second*
  complete set of chart detections in the store the report reads from. The client's monthly detection
  count is now inflated by content they already had processed once — not a rounding error, a real
  double-count that misrepresents actual document volume and, worse, can double-queue the same charts
  into a downstream MLR-review workload that has to be triaged twice for no reason.
- **No model-version stamp** → a month-over-month spike or dip in detection counts becomes
  fundamentally unattributable. Nobody reading the report can tell "the client sent more documents
  this month" apart from "we retrained the model in May and it's now more (or less) sensitive to
  forest plots" — and guessing wrong in either direction is costly: concluding a real client-volume
  spike is a model regression wastes engineering time chasing a phantom bug, while concluding a real
  model-driven sensitivity change is just "more documents" hides an actual, worth-knowing-about shift
  in model behavior from whoever owns the retraining cadence.

Both failure modes share the same shape as course 03's retrieval tie-in: **the fix has to reach as far
downstream as the data itself does.** A `content_hash`/`model_version`-aware `detection_runs` table
solves the bookkeeping at the point of inference, but a reporting layer built before that table existed
would need to be updated to actually *use* `supersedes_run_id` (exclude superseded runs from counts)
and `model_version` (segment trend charts by model version, or at minimum annotate a version-change
event on the timeline) — otherwise the correct data exists upstream while the report consuming it keeps
producing the same misleading numbers it always did.

## The full interview-ready answer, assembled

> "Today, nothing in this pipeline checks whether a document's content has been seen before, and no
> detection result carries a stable model-version identifier — every inference call is independent,
> and the only version-adjacent field in the response is a cold-start timestamp, which says nothing
> about which model weights actually produced the boxes. I want to be upfront that, unlike some of the
> other projects in this portfolio, there's no source repository behind this one to confirm that
> against — this is my best technically grounded reconstruction of how a pipeline shaped like this one
> would behave, and how I'd close the gap, not a verified account of shipped code.
>
> What you could do today with existing AWS primitives is a manual workaround: S3 already gives you a
> content checksum on every object, and CloudWatch Logs can be queried to find a prior invocation with
> matching content, if you're logging that — the pieces exist, they're just not wired together for
> this purpose yet.
>
> The two concerns worth keeping separate: whether a document is a corrected resubmission of something
> already processed is a deduplication/supersession question, about the document's identity over time.
> Whether two detection results came from the same model is a completely different, attribution
> question, about the model's identity over time. A reprocessed document showing different detections
> could mean a duplication bug, a legitimate document correction, or an expected model-version change
> — and conflating those three explanations is exactly the mistake to avoid.
>
> If I were building the real fix, I'd have the orchestration layer compute a content hash before ever
> calling the detection endpoint, keep a small `detection_runs` table keyed by that hash plus a logical
> document ID, and stamp every result with the actual deployed model's version — so an exact repeat
> short-circuits without paying for inference again, a corrected resubmission gets linked to what it
> supersedes instead of sitting as an unrelated duplicate, and a model retrain is always attributable
> instead of looking like unexplained noise. And I'd make sure that fix reaches the downstream reporting
> layer too — a per-client detection-count dashboard needs to actually exclude superseded runs and
> segment by model version, or the correct data sitting upstream doesn't stop the report from telling
> the wrong story."

## Tying It Back

The strongest part of this answer isn't the proposed `detection_runs` design in Part 4 — it's Part 3:
drawing a clean line between "has this document been seen before" and "which model produced this
result," so an interviewer can see those aren't being treated as the same problem under pressure. As in
course 03, being precise about what's *not* handled today, and exactly why two adjacent-sounding
concerns are actually different axes, is as strong a signal as the design itself.
