# 99 — Interview Q&A: Chart/Graph Detection

Read this last, once Chapters 00–07 are fresh. **Q1 is the gotcha question** — the kind of
systems-level follow-up an interviewer asks after hearing "fine-tuned model, retrained periodically,
deployed via Lambda, running in production for two clients" — it comes first, deliberately, ahead of
the general behavioral warm-up, the same way course 05's real interview question leads its own Q&A.
Everything after is grouped behavioral → technical → system design → retrospective → production
context, though in a real interview they'll be mixed together, often as follow-ups to each other.

---

## Q1. If a corrected or resubmitted version of a document comes through again, how do you avoid double-counting its chart/graph detections — and how do you keep results comparable after the model gets retrained?

The full worked answer lives in Chapter 06
(`06-detection-deduplication-and-model-version-drift.md`) — this is the compressed, interview-ready
version. **One honesty note up front, unlike course 05's Q1:** there's no source repository behind
this project the way there is for the Document Uploader Service, so this answer is presented as a
technically grounded reconstruction of how a pipeline shaped like this one would behave and how I'd
close the gap — not a verified account of shipped code, and worth saying so if asked directly.

> "Today, nothing in this pipeline checks whether a document's content has been seen before, and no
> detection result carries a stable model-version identifier — every inference call through the Lambda
> handler is independent, and the only version-adjacent field in the response is a cold-start
> timestamp, which changes on every container recycle regardless of whether the underlying model
> weights changed at all. So two different failure modes can happen silently: a resubmitted or
> corrected document gets processed twice, producing two unrelated sets of detections instead of one
> current set superseding the other; and a retrained model gets deployed with no way to trace a change
> in detection output back to 'the model changed' versus 'the client sent different documents.'
>
> Those are two genuinely different concerns, and I'd be careful not to conflate them: whether a
> document is a corrected resubmission is a question about the *document's* identity over time —
> that's a deduplication/supersession problem. Whether two results came from the same model is a
> question about the *model's* identity over time — that's an attribution problem, completely
> orthogonal to whether the document changed. A reprocessed document showing different detections could
> mean a duplication bug, a legitimate document correction, or an expected model-version change, and
> jumping to the wrong explanation is the mistake to avoid.
>
> What you could do today with existing AWS primitives is a manual workaround — S3 already computes a
> content checksum on every object, and CloudWatch Logs can be queried for a prior invocation with
> matching content if you're logging that. The pieces exist; they're just not wired together for this
> purpose yet.
>
> If I were building the real fix, I'd have the orchestration layer compute a content hash before ever
> calling the detection endpoint, keep a small `detection_runs` table keyed by that hash plus a logical
> document ID, and stamp every result with the actual deployed model's version. An exact repeat
> short-circuits without paying for another inference call; a corrected resubmission gets linked to
> what it supersedes instead of sitting as an unrelated duplicate; a model retrain is always attributable
> instead of looking like unexplained noise. And that fix has to reach the downstream reporting layer
> too — a per-client detection-count dashboard needs to exclude superseded runs and segment by model
> version, or the correct data sitting upstream doesn't stop the report from telling the wrong story."

**Follow-ups to be ready for:**

- *"So a client resubmitting the same file twice today just produces two independent results?"* —
  Yes, and worth saying plainly: there's no content check, no locking, nothing. It's exactly the kind
  of silent duplication that only surfaces once someone downstream notices a report doesn't add up.
- *"Would you say that's a gap, or was it never in scope?"* — Frame it like course 05's equivalent
  answer: it's a gap that was never in scope for what the resume bullet describes building (labeling,
  fine-tuning, deployment), not an oversight discovered after the fact — naming it as "not yet built"
  rather than either overclaiming it's handled or apologizing for it.
- *"Where would you actually compute the content hash — in the Lambda, or somewhere else?"* — Not in
  the Lambda handler; it should stay stateless and fast, which is the whole point of Chapter 04's
  serverless argument. The hash lookup belongs in the ECS Fargate orchestration layer, which already
  owns cross-cutting, stateful concerns like per-client routing (Chapter 04) — a DynamoDB table keyed
  by content hash is a natural, low-latency fit there.
- *"How would you prioritize building that versus other things?"* — Depends on how often documents are
  genuinely resubmitted/corrected and how often the model is actually retrained — worth quantifying
  with real pipeline telemetry before committing engineering time to the two-part fix (dedup table +
  version stamping) described above, the same "quantify before prioritizing" answer course 05 gives for
  its own versioning gap.

---

## Behavioral

### Q2. Tell me about a computer vision project you're proud of.

**(STAR)** At Indegene, pharma slide decks and scientific documents embed charts and graphs —
efficacy curves, comparison bar charts, forest plots — directly as image regions mixed in with body
text, and downstream document-processing pipelines needed to know which regions were charts so they
could be routed separately rather than run through text-oriented processing. I ran a collaborative
labeling effort on AWS Ground Truth to build a custom bounding-box dataset, fine-tuned a YOLOv5 object
detector on that dataset using AWS Sagemaker (starting from COCO-pretrained weights), and deployed
the trained model behind a Lambda function fronted by API Gateway so any pipeline stage could call a
simple REST endpoint. The result was a model that attained a 96% True Positive Rate at a strict 0.85
IOU threshold — meaning it found nearly every chart on a page, with boxes tight enough to safely crop
and hand off to the next processing stage. I'm proud of it because it's a full pipeline I owned
end-to-end: data labeling process, model training, and production deployment, not just a notebook
experiment.

### Q3. Describe a time you had to make a tradeoff between accuracy and system constraints.

Deploying the fine-tuned YOLOv5 model behind Lambda meant balancing model size/accuracy against
cold-start latency and Lambda's memory/timeout ceiling. A larger input image size (e.g., 1280 instead
of 640) improves detection of small, fine-detailed charts, but increases both inference latency and
memory footprint on a resource-constrained Lambda execution environment. I had to reason about this
as a genuine tradeoff rather than always maximizing accuracy — sizing memory and image resolution
empirically against real latency budgets, and considering Provisioned Concurrency to absorb baseline
latency-sensitive traffic while letting normal Lambda scaling handle burst overflow at the cost of
occasional cold starts.

### Q4. Tell me about a time you had to resolve disagreement within a team on a technical process.

Collaborative labeling on AWS Ground Truth meant multiple annotators drawing bounding boxes around
chart regions, and annotators don't always agree on exactly where a box should start and end —
whether to include a legend, how much margin to leave, whether a borderline table-with-a-trendline
counts as a chart. The fix isn't to have one person unilaterally decide; it's process: clear labeling
instructions with visual right/wrong examples up front, measuring inter-annotator agreement via IOU
on overlapping samples, and routing genuinely ambiguous cases to a senior reviewer for adjudication
rather than letting inconsistent labels silently degrade the training data.

---

## Technical Deep-Dive

### Q5. Why did you choose an IOU threshold of 0.85 instead of the standard 0.5?

IOU=0.5 is the common default in general object-detection benchmarks because it only requires "roughly
in the right place," which is fine when a human is the final consumer of the detection. Here, the
detection output is meant to feed an automated downstream step that likely crops the image at the
predicted box for further processing — a loosely-fit box at 0.5 IOU could clip part of the chart's
legend or include adjacent body text in the crop, corrupting whatever runs next. Evaluating (and by
extension, optimizing) at 0.85 is a direct proxy for "is this box tight enough to safely automate a
crop on," which is the real production requirement, not just "did the model notice a chart existed."
It's a stricter, more production-relevant bar, and hitting 96% TPR at that bar is a meaningfully
stronger result than the same number would be at 0.5.

### Q6. Walk me through IOU and how it's computed.

IOU (Intersection over Union) is the overlap area between a predicted box and the ground-truth box,
divided by the union area of the two boxes — `intersection / union`. It's bounded between 0 (no
overlap) and 1 (identical boxes). Computing it requires finding the overlapping rectangle's
coordinates (max of the two left/top edges, min of the two right/bottom edges), clamping to zero if
there's no overlap, then dividing that intersection area by `area_a + area_b - intersection`. See
`notebooks/03_iou_map_metrics_from_scratch.ipynb` for a from-scratch numpy implementation and a worked
"TPR at IOU=0.85" computation on synthetic boxes.

### Q7. What's the difference between TPR/recall and precision, and why report TPR here specifically?

Recall (TPR) is "of all real charts, what fraction did the model find" — `TP / (TP + FN)`. Precision
is "of all boxes the model predicted, what fraction were real charts" — `TP / (TP + FP)`. They're
independent axes; a model could have high recall and mediocre precision (finds everything but also
over-triggers) or the reverse. Reporting TPR alone is a recall-first framing, appropriate for a triage
use case where missing a real chart means silently skipping important pharma content downstream — a
false positive is comparatively cheap (the downstream stage looks at a region that turns out not to
be a chart), while a false negative means content is never routed for the processing it needed. That
said, a complete picture would also track precision or a business proxy for it (e.g., manual review
rate); TPR alone is one half of the story, and I'd flag that explicitly to a stakeholder relying on
just this one number.

### Q8. How does mAP relate to what you reported, and why didn't you report mAP instead?

mAP averages Average Precision (area under the precision-recall curve) across classes, and often
across a sweep of IOU thresholds (COCO's mAP@0.5:0.95). It's the right metric for comparing detector
architectures across a broad operating range. TPR at a single, strict, fixed IOU threshold is a
narrower, more directly business-relevant metric here: it answers exactly the question that matters
for this triage use case — "at the tight localization bar our downstream cropping needs, how often do
we find the real chart" — without needing to interpret a curve or an averaged-across-thresholds
number. I'd be comfortable computing and reporting mAP too if asked to benchmark against another
architecture, but TPR@IOU=0.85 is the more actionable number for this specific deployment decision.

### Q9. Why YOLOv5 instead of a two-stage detector like Faster R-CNN?

YOLOv5 is a single-stage detector — it predicts boxes and classes in one forward pass, versus
two-stage detectors that first propose candidate regions and then classify/refine each one
separately. Single-stage detectors are faster at inference and have a smaller model footprint, which
matters directly for a Lambda deployment where model artifact size affects cold-start load time and
memory footprint, and where inference latency is part of the request-response time a caller is
waiting on. Two-stage detectors can edge out single-stage detectors on raw accuracy in some
benchmarks, but for a bounding-box task on a fairly structured, visually distinct object class like
charts — not a benchmark with hundreds of visually similar fine-grained classes — YOLOv5's speed and
deployability tradeoff was the right call, and the 96%-TPR result suggests it wasn't leaving
meaningful accuracy on the table.

### Q10. How do you handle disagreement between AWS Ground Truth labelers?

First line of defense is prevention: clear, example-driven labeling instructions that remove
ambiguity before annotators start (what counts as the chart's boundary, whether the legend/title is
included, what's explicitly *not* a chart — e.g., a table). Second, measure it: have overlapping
annotators label a shared subset and compute IOU between their boxes on the same objects — a low
average agreement signals an instructions or class-definition problem, not just "a labeler is being
careless." Third, resolve it: route genuinely disagreeing or ambiguous cases to a senior
reviewer/adjudicator, who either picks the more correct annotation, draws a corrected box, or excludes
the instance entirely if it's a genuinely ambiguous edge case that would just inject label noise. I'd
also spot-check a random or risk-weighted sample of "agreed" annotations rather than assuming
agreement always equals correctness.

### Q11. How would you reduce Lambda cold-start latency for a YOLO model?

A few concrete levers, roughly in order I'd reach for them: (1) make sure the model is loaded once at
module scope outside the handler, so cold-start cost is paid once per container and every subsequent
warm invocation reuses it; (2) use Provisioned Concurrency to keep a baseline number of environments
pre-warmed with the model already loaded, for latency-sensitive traffic; (3) shrink the model
artifact and its dependency footprint — export to ONNX Runtime or TorchScript instead of shipping the
full PyTorch training-time stack, which reduces both image size and load time; (4) right-size Lambda
memory, since Lambda scales CPU with memory and a CPU-bound inference workload can get faster, not
just more expensive, at a higher memory tier — verified empirically, not assumed. If none of that gets
latency low enough for the traffic pattern, that's the signal to reconsider Lambda entirely and move
to an always-warm Sagemaker real-time endpoint instead (Chapter 04). Worth naming too: if the timeout
itself was simply left at Lambda's 3-second platform default instead of being sized against real
cold-start cost, that's not a latency-tuning problem at all — that's the config bug in Chapter 07,
and no amount of the above fixes it until the timeout is actually configured.

### Q12. Explain anchor boxes and why they matter for YOLOv5's predictions.

Anchor boxes are predefined width/height reference shapes at each detection scale; instead of
predicting a box's dimensions from scratch, the model predicts a small offset relative to the nearest
anchor, which is an easier, more stable target to learn than raw coordinates. Anchors that don't match
the actual aspect-ratio distribution of your objects hurt convergence and accuracy — YOLOv5 can
auto-recompute anchors from the training set's actual box shapes at the start of fine-tuning, which
matters here because chart bounding boxes cluster around wider, landscape-style aspect ratios than
COCO's general-object anchor set (tuned for people, cars, animals).

### Q13. GANs are notoriously unstable to train — why, and how would you handle it in SRGAN?

There's no single loss curve that monotonically signals "getting better" because the generator and
discriminator are in a competitive dynamic — the generator's loss rising can mean the discriminator
is legitimately improving, not that the generator regressed. Two specific failure modes: mode
collapse, where the generator finds a narrow set of outputs that reliably fool the current
discriminator instead of learning the full diversity of realistic outputs; and gradient
vanishing/imbalance, where if either network gets too far ahead of the other, the losing network's
gradient signal becomes uninformative and training stalls. Standard mitigations: pretrain the
generator with pixel-wise loss first so it starts from a reasonable content baseline before
introducing the harder adversarial signal; keep a perceptual/feature-space loss term alongside the
adversarial loss as a stabilizing anchor so the generator doesn't drift into pure hallucination;
balance the training pace/learning rates between generator and discriminator; and monitor qualitative
sample outputs throughout training, since the loss curves alone are a poor signal of GAN training
health.

### Q14. Why does adversarial/perceptual loss produce sharper super-resolution results than pixel-wise MSE?

MSE is minimized by the statistical average of all plausible high-frequency detail that could explain
a blurry low-res patch, so a pure-MSE-trained model hedges toward a safe, blurry average rather than
committing to one sharp, plausible reconstruction — fine texture gets smeared out. Adversarial loss
instead rewards the generator for producing outputs the discriminator can't distinguish from real
high-resolution images — pushing toward the statistical manifold of *realistic* images rather than
the pixel-wise average. SRGAN pairs this with a perceptual loss computed in a pretrained network's
feature space (rather than raw pixels), so two images that differ slightly in exact pixel values but
match in higher-level texture/content score as similar. The combined effect directly targets what a
human actually judges as "sharp," which raw pixel MSE doesn't.

### Q15. Where could AWS Rekognition have substituted for part of this project, and where would it fall short?

Rekognition Custom Labels is a fully managed computer-vision training/inference service — no
Sagemaker training job, no Lambda packaging, no cold-start tuning required. For a quick
proof-of-concept, or a use case with a looser accuracy bar, it's worth prototyping before investing in
a custom pipeline. It falls short here because hitting a specific, strict target — 96% TPR at IOU=0.85
— needs fine-grained control over things like anchor box shapes, augmentation strategy, and
IOU-threshold-tuned evaluation that a managed black-box service doesn't expose, and per-call pricing
at high volume can exceed the cost of a self-hosted model. The honest answer is "great for
prototyping or a lower bar, not a substitute when you need to hit and prove a specific strict
accuracy target."

---

## System Design

### Q16. Design a real-time chart-detection API that must handle bursty traffic cost-effectively.

Key requirements to surface before designing: what's "real-time" in latency terms, how bursty
("bursty" could mean 10x spikes a few times a day, or highly unpredictable second-to-second), and
what's the acceptable cost-per-inference ceiling.

A reasonable design: **API Gateway** as the public entry point (auth, throttling, request validation)
in front of **Lambda running a container-image-packaged YOLOv5 model** for the bulk of traffic — this
gets automatic scale-to-zero-and-back-up behavior, so idle periods cost nothing. To handle the
latency-sensitive tail without over-provisioning, use **Provisioned Concurrency** sized to typical
baseline load (not peak), so common-case requests avoid cold starts, while burst overflow beyond that
baseline falls back to normal (cold-start-paying) Lambda scaling — a deliberate choice to eat
occasional burst-time latency rather than pay for peak capacity 24/7. Add **SQS as a buffer** in front
of Lambda if the API's callers can tolerate asynchronous processing (submit an image, poll or get a
webhook callback for results) rather than strict synchronous request-response — this smooths bursts
into a steady processing rate and lets Lambda concurrency stay lower and cheaper, at the cost of not
being purely "real-time" request-response. If sustained volume analysis later shows traffic isn't
actually bursty but has grown into a high, steady baseline, that's the trigger to migrate the hot path
to a Sagemaker real-time endpoint or TensorFlow Serving fleet (Chapter 04) instead of continuing to
pay Lambda's per-invocation premium at high, predictable volume — cost-effectiveness is a moving
target that should be revisited as traffic patterns become clearer, not a one-time architecture
decision.

Monitoring worth building in from day one: CloudWatch on Lambda concurrency/throttling/duration/error
rate, API Gateway 4xx/5xx and latency percentiles, and a model-quality dashboard tracking detection
confidence distribution over time (a proxy for silent data drift — e.g., a client starts sending a
chart style visually different from the training distribution).

---

## Retrospective

### Q17. What would you change if you rebuilt this today?

A few concrete things: First, I'd invest more in the active-learning loop from day one rather than
labeling in an arbitrary order — using an early model's low-confidence predictions to prioritize what
gets labeled next would likely have hit the same accuracy bar with meaningfully less total annotation
effort. Second, on deployment, I'd benchmark ONNX Runtime or TorchScript export from the start rather
than shipping a full PyTorch inference stack in the Lambda container — smaller artifact, faster cold
starts, same accuracy. Third, I'd track precision/false-positive rate alongside TPR from the
beginning as a paired metric, and build a lightweight feedback loop (even just periodic manual review
of a sample of low-confidence detections) to catch data drift as client document styles evolve over
time, rather than relying on the initial evaluation number staying representative indefinitely.
Fourth — and this is the pair of fixes Chapters 06 and 07 lay out in depth — I'd build the
content-hash dedup check and `model_version` stamping into the orchestration layer from the start,
and read the model artifact location from an environment variable instead of baking it into the
container image, so a retrain is a config change with a traceable version tag rather than a rebuild
with no attributable identity. Finally, for the SRGAN side project specifically, I'd extend the
frame-by-frame approach with a temporal-consistency mechanism (optical flow or a recurrent component
across frames) to remove the subtle flicker that independent per-frame super-resolution introduces in
video output — a known limitation of the simple frame-by-frame design that a genuinely video-aware
architecture would address directly.

---

## Client & Production Context

### Q18. This ran in production for two pharma clients, Eli Lilly and AstraZeneca — why did missing a chart/graph detection actually matter to them operationally, not just as a benchmark metric?

Because this sat at the *front* of a larger, real content pipeline those clients depended on, not in
isolation. A missed detection meant a document — a slide deck page, a clinical report page — moved
forward through the pipeline without ever being flagged for the chart-extraction/routing step it
needed: no bounding box means no crop, no crop means the downstream chart-digitization or MLR-review
stage never sees that region at all. Concretely, that could mean an efficacy curve or a forest plot
silently never gets pulled into structured indexing, or a page that should have been routed for
chart-specific regulatory review instead flows through as if it were plain text — and because it's
*silent*, nobody gets an error to investigate; the content just quietly doesn't get the processing it
needed until someone downstream notices a gap. For a paying client's production content pipeline,
that's a real operational miss, not an abstract recall number — which is exactly why we evaluated at a
strict IOU=0.85 bar and optimized for TPR/recall first (Q5, Q7): a benchmark team can shrug off "96%
instead of 98%," but a client whose Kaplan-Meier curve got missed on a real submission document
can't.

### Q19. How did you keep Eli Lilly's and AstraZeneca's documents and detection results isolated on AWS while sharing the same fine-tuned model?

The model and inference code path were shared — there's no accuracy or business reason to train and
maintain two separate models for what's the same detection task — but the *data* was isolated at the
storage and access-control layer, not the model layer. Documents and detection outputs for each client
lived under separate **S3 prefixes** (effectively separate logical buckets within the same account
structure), and access to each prefix was scoped through **IAM policies** tied to that client's
processing role, so the ECS Fargate orchestration component (or Lambda execution role) handling Eli
Lilly's documents simply has no permissions that would let it read or write into AstraZeneca's prefix,
and vice versa — enforced by AWS, not by application-level convention that a bug could quietly violate.
The Lambda/Sagemaker inference call itself is stateless and client-agnostic — it receives an image,
returns bounding boxes, and never persists anything — so the isolation boundary really only has to be
enforced at the two places state actually lives: where documents are read from, and where results are
written back to. That's a deliberately narrow, auditable isolation boundary rather than trying to bolt
multi-tenancy logic into the model or the inference handler itself.

### Q20. How would you handle a sudden document-volume spike from one client without starving the other client's processing SLA, given the Lambda + Sagemaker endpoint architecture?

A few layers, from cheapest to most involved. First, Lambda's concurrency itself is a shared,
account-level resource, so an unthrottled spike from one client can legitimately starve the other's
requests of available execution environments — the fix is **reserved concurrency** (or a Lambda alias
with its own concurrency limit) per client, so a burst from Eli Lilly is capped at its own ceiling and
physically cannot consume all the concurrency AstraZeneca's traffic also needs. Second, front the
Lambda call with **per-client throttling at API Gateway** (usage plans / API keys with their own rate
limits), so the orchestration layer enforces fairness before a spike even reaches Lambda. Third, if the
spike is a large batch rather than latency-sensitive real-time traffic — which most volume spikes in a
document pipeline actually are — route it through an **SQS buffer** from the ECS Fargate orchestration
layer instead of calling the synchronous endpoint directly, so the spike gets smoothed into a steady
processing rate against both clients' SLAs rather than competing for the same real-time capacity;
Provisioned Concurrency reserved for baseline/latency-sensitive traffic stays protected either way. If
sustained volume from either client grows enough that this becomes a recurring problem rather than an
occasional spike, that's the signal (Chapter 04) to move the hot path from Lambda to a dedicated,
auto-scaled Sagemaker real-time endpoint fleet with its own per-client capacity, rather than continuing
to solve it with throttling and queuing on top of a shared serverless concurrency pool. Chapter 07 has
the fuller version of this caveat, including illustrative reserved-concurrency and Provisioned
Concurrency figures per client.

### Q21. Walk me through a bug in an object-detection deployment that a model-accuracy test on its own validation set would never catch.

The sharpest example is the train/serve preprocessing mismatch in Chapter 07: YOLOv5's training
pipeline resizes images with **letterbox resizing** (scale to fit, preserve aspect ratio, pad the rest)
because every training-time bounding box is labeled relative to that geometry. An early Lambda handler
instead did a naive aspect-ratio-distorting stretch-resize — easy to write, doesn't throw an error, and
produces systematically shifted, undersized boxes on any non-square input, which is the common case for
landscape slide images. Re-running the model's own validation-set evaluation (Chapter 03) would never
surface this, because that evaluation never touches the serving code's preprocessing at all — it only
exercises the training pipeline's own data loader. The only way to catch it is a test that runs a fixed
input image through the *actual deployed serving path* and compares it against the training-side
transform directly, or periodically re-running the IOU/TPR evaluation against real production inputs
end-to-end rather than only against the held-out training-time validation split.

### Q22. You mentioned an IOU threshold of 0.85 for evaluation — is that the same IOU threshold used somewhere else in the pipeline, and would mixing them up matter?

No, and this is a real, easy-to-conflate pair worth naming precisely. The 0.85 threshold (Chapter 03)
is an **evaluation-time** definition of "does this prediction count as correct" — it's never applied
inside the model itself. Separately, YOLOv5 applies **Non-Maximum Suppression** at inference time using
its own IOU threshold (illustratively 0.45, Chapter 07) to decide whether two overlapping candidate
boxes represent the same object and should be merged into one, versus two genuinely separate objects.
Turning the NMS threshold up or down changes how many raw candidate boxes survive into the final
output; it has nothing to do with how the evaluation harness later judges whether any of those surviving
boxes counts as a correct detection. Mixing them up in an interview answer — implying the model was
"trained" or "tuned" to 0.85 — is a specific, avoidable mistake; the honest, precise version keeps the
two firmly separate.

### Q23. If a client's monthly chart-detection count suddenly spikes, how do you figure out whether that's the model or the documents?

Today, illustratively, you mostly can't — cleanly, at least — which is exactly the gap Chapter 06 is
built around and exactly the failure mode in Chapter 07's bug #3 (a retrained model shipped with no
version tag anywhere in the response payload or deployment metadata). The fix isn't a one-off
investigation, it's making the question answerable by construction: stamp every detection result with
the actual model version that produced it, and segment any trend report by that version rather than
treating detection counts as one undifferentiated time series. With that in place, a spike coinciding
with a model-version change is directly attributable to the retrain; a spike with no version change
alongside it is attributable to real document volume. Without it, the honest answer to "is this the
model or the documents" is a guess dressed up as an investigation — and Chapter 07 also names why that
gap exists today: the model artifact's location is baked into the container image with no version
identifier read from configuration, so there was never a natural place to stamp that tag in the first
place.
