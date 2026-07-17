# 99 — Interview Q&A

Read this last, once Chapters 00–05 are fresh. Questions are grouped behavioral -> technical ->
system design -> retrospective, roughly the order a real interview tends to escalate.

---

### 1. (Behavioral, STAR) Tell me about a project where you significantly improved a system's accuracy.

**Answer:** At Indegene, I worked on automated citation tracking for pharma marketing and scientific
documents — slide decks, leave-behinds, journal reprints — where every claim needs a traceable
superscript citation marker for regulatory/MLR compliance. The starting approach, essentially OCR
text plus a regex heuristic to guess which digits were citations, was only about **5% accurate**,
because "is this a citation marker" is fundamentally a visual/positional question (small size,
raised position) that flat OCR text can't answer. I designed a sequential three-stage pipeline
instead: OCR for text and geometry, a YOLO object detector to find small, elevated candidate
regions, and a Deep-CNN classifier to make the final true/false call on each candidate and filter
out look-alikes like page numbers and exponents. That architecture took citation tracking accuracy
from **5% to 85%**. The key lesson I took from it: when a heuristic is failing badly, it's worth
asking whether you're solving the actual problem (visual/positional classification) or a proxy for
it (text pattern matching) — the fix here wasn't a better regex, it was recognizing the problem
needed a different kind of model entirely.

---

### 2. (Behavioral) Why did you choose a multi-stage pipeline instead of one end-to-end model?

**Answer:** Because the three sub-problems — general text/layout extraction, small-object
localization, and fine-grained visual classification — have different data requirements, different
appropriate loss functions, and very different amounts of labeled data realistically available for
each. Training one model to do all three jointly means a much harder, less sample-efficient
optimization problem, and it makes debugging nearly impossible — if the end-to-end model gets it
wrong, you have no way to tell whether it failed to *find* the text, failed to *localize* the
candidate, or failed to *classify* it. Three focused models, each evaluated against its own
appropriate metric (Chapter 04), let me isolate exactly where errors were coming from and improve
that stage specifically.

---

### 3. (Technical) Why not just use OCR font-size heuristics instead of a full detection+classification pipeline?

**Answer:** That's essentially what the ~5% baseline was, and it fails for a few concrete reasons.
First, most OCR output doesn't preserve font size and vertical baseline offset as clean, per-glyph
fields you can threshold on reliably — segmentation errors mean a superscript can get merged into
the preceding word, or dropped as noise, before you ever get a usable size signal. Second, even
where you do get a size signal, "small and slightly raised" alone isn't a sufficient rule — page
numbers, list markers, exponents, and subscripts all share that visual signature, so a
threshold-based heuristic has a structurally high false-positive rate. Third, scanned/photographed
documents introduce skew and noise that corrupt baseline estimation itself, so even a
well-calibrated threshold on one document can be wrong on the next. A learned detector
(YOLO) plus a learned classifier (CNN) can pick up on more subtle, higher-dimensional visual cues
than a hand-tuned size/position threshold ever could, and they generalize across the document-source
diversity a rule-based heuristic doesn't.

---

### 4. (Technical) How do you handle class imbalance when superscripts are a tiny fraction of all characters?

**Answer:** This shows up at both Stage 2 and Stage 3. For the YOLO detector, I'd make sure the
training set includes deliberately-curated hard negatives — page numbers, exponents, subscripts,
tightly-kerned punctuation — not just random background, so the detector learns the actual
discriminating boundary rather than just "small things are rare, ignore them" or the opposite
failure of over-triggering on everything small. For the CNN classifier, since it only ever sees
YOLO's candidates (which are already a much more balanced true/false mix than the raw document, by
construction — recall Chapter 04's "narrowing funnel" point), the imbalance problem is naturally
less severe than it would be if the classifier ran over the whole page. Where imbalance remains, the
standard toolkit applies: class-weighted loss (upweighting the minority "true superscript" class),
oversampling true positives or augmenting them (slight rotation/scale jitter, since these are
visual, position-sensitive objects), and evaluating with precision/recall/F1 rather than raw
accuracy, since accuracy is a misleading metric on an imbalanced binary task — a classifier that
always predicts "false positive" could still show high raw accuracy while being useless.

---

### 5. (Technical) Walk me through IOU and non-max suppression — why does YOLO need NMS at all?

**Answer:** IOU (intersection over union) is the overlap ratio between two boxes — intersection area
divided by union area, 0 to 1. YOLO needs NMS because its grid-based design means multiple nearby
grid cells (and multiple anchors within a cell) will often all fire with reasonable confidence
around the same true object, producing several overlapping candidate boxes for one physical
superscript. NMS resolves that: sort candidates by confidence, keep the highest-confidence box,
discard any remaining box that overlaps it above an IOU threshold (since that's almost certainly a
duplicate detection of the same object, not a second real object), and repeat. Without NMS, a single
true superscript could get reported to Stage 3 multiple times as separate candidates, which would
distort both the classifier's throughput and the final citation count. I implemented both IOU and
NMS from scratch in numpy — walking through that is in
`notebooks/02_yolo_object_detection_demo.ipynb`, and I'd be glad to talk through the code directly.

---

### 6. (Technical) YOLOv5 vs. newer YOLO versions (v8, v10, etc.) — what are the trade-offs?

**Answer:** The core single-shot, grid-based idea is shared across the family; what's changed
version to version is mostly the detection head, the anchor strategy, and training-time
augmentation/loss tricks. YOLOv5 (and earlier) use predefined anchor boxes, which means an explicit
anchor-tuning step (e.g., k-means clustering on the training set's box dimensions) — for a narrow,
consistent target shape like superscript crops, that's not a huge burden, but it is one more thing
to get right and re-tune if the target object's size distribution shifts. YOLOv8 onward moved to an
**anchor-free** detection head, predicting box centers and extents directly, which simplifies the
pipeline and tends to generalize a bit better without manual anchor tuning — relevant if this system
were rebuilt today. Newer versions also generally bring better small-object performance through
improved multi-scale feature fusion, which is directly relevant here given how small a superscript
character is. The trade-off to weigh in practice: newer versions aren't always a strict upgrade for
a narrow, already-well-scoped task like this one — YOLOv5's maturity, broad tooling support, and the
fact that this task's candidates are pre-cropped to text-line regions anyway (mitigating a lot of
the small-object disadvantage per Chapter 02/05) mean the marginal benefit of chasing the newest
version needs to be weighed against retraining/reannotation cost and integration effort in a
production pipeline that's already working.

---

### 7. (Technical) Why add a CNN classifier after YOLO instead of just improving YOLO's own classification head or raising its confidence threshold?

**Answer:** Raising YOLO's confidence threshold directly trades recall for precision *inside a
single model that's also responsible for localization* — you can't tune that trade-off
independently for classification without also hurting its ability to find true candidates in the
first place, since both come out of the same forward pass and the same training objective.
Splitting the two concerns into separate models lets each be tuned for its own job: YOLO stays
biased toward recall (better to over-generate candidates), and a separate CNN, seeing only
pre-cropped, pre-localized candidates, can afford to be trained and evaluated purely on
precision for the true/false discrimination — with its own dedicated architecture, loss, and
training data (deliberately including hard negatives), which a shared detection+classification head
can't specialize on to the same degree.

---

### 8. (Technical) How would GPT-4 Vision (or a general vision-language model) change this pipeline if you rebuilt it today?

**Answer:** A modern VLM like GPT-4V could plausibly collapse Stages 2 and 3 (and arguably part of
Stage 1) into a single prompted call: "here's a cropped text-line image, identify any citation-style
superscript markers and their positions/values." That would meaningfully cut engineering
complexity — no separate YOLO training/annotation pipeline, no separate CNN training pipeline — and
VLMs are strong at exactly this kind of "reason jointly about visual layout and text semantics" task
that was awkward to split across three purpose-built models in 2021-2023 when this was likely built.
The trade-offs: **cost and latency** per call are far higher than a lightweight YOLO+CNN pass
running at scale over a large document corpus (this is the crux of the system-design question
below), **determinism/consistency** is harder to guarantee with a generative model than with a
calibrated classifier's probability output, and there's a real question of whether a general-purpose
VLM's accuracy on this specific, narrow, visually-subtle distinction would actually beat a model
trained specifically on it without careful prompt engineering and few-shot examples of true vs.
false-positive superscripts. A realistic modern architecture is probably a **hybrid**: keep the
cheap OCR+YOLO funnel to narrow down candidates at low cost and high recall, and use a VLM only as
the final-stage classifier/adjudicator on the (now small) set of ambiguous candidates, or as an
offline QA/audit tool rather than the on-path production classifier. This connects to the
**Transformers / GPT-4 Vision** line in the skills list — the realistic story is evaluating and
prototyping with a VLM as part of exploring this trade-off, not necessarily replacing the production
pipeline wholesale on day one.

---

### 9. (Technical) How is Sagemaker used in a pipeline like this?

**Answer:** AWS Sagemaker is a natural fit for both training and hosting the two learned models here.
For training: Sagemaker Studio/Training Jobs give managed, scalable compute for training the YOLO
detector and the CNN classifier without managing GPU infrastructure directly, with experiment
tracking across hyperparameter runs. For labeling: **Sagemaker Ground Truth** is a reasonable choice
for the bounding-box annotation work Chapter 02 describes, especially given how specialized the
annotation guidelines need to be (tight boxes on tiny objects, careful hard-negative labeling). For
serving: Sagemaker endpoints can host the trained YOLO and CNN models behind a low-latency inference
API, which matters if this pipeline needs to run as part of a near-real-time document-ingestion
workflow rather than purely as an offline batch job.

---

### 10. (Technical) What's the difference between text detection and text recognition in OCR, and why does that distinction matter here?

**Answer:** Detection finds *where* text is (bounding boxes/regions); recognition decodes *what* the
text says, given a region known to contain it. The distinction matters here because Stage 1's
detection output — word/line bounding boxes and baseline metadata — is the geometric scaffold Stage
2's YOLO model operates on; the actual recognized text content matters less to Stage 2 than the
geometry does. If I only had OCR's recognized text string with no positional metadata, I'd have
nothing for YOLO to search near, and the whole "look for a small, elevated region relative to this
line's baseline" strategy wouldn't be possible — which is exactly why Chapter 01 emphasizes that
OCR's contribution to this pipeline is geometry as much as it is text.

---

### 11. (System Design) Design this to run at scale over millions of pages with acceptable latency and cost.

**Answer:** I'd treat this as a batch/offline pipeline rather than a synchronous request-response
system, since citation tracking doesn't need sub-second latency per page — a few seconds to
minutes of end-to-end processing per document is acceptable if throughput is high. Concretely:

- **Ingestion & queueing**: incoming documents land in S3; an event (S3 trigger -> SQS/EventBridge)
  fans out per-page processing jobs so pages can be processed independently and in parallel.
- **Stage 1 (OCR)**: for a corpus this size, a managed service like **AWS Textract** removes the
  operational burden of scaling a self-hosted OCR fleet and gives built-in autoscaling, at a
  per-page cost that needs to be weighed against self-hosting Tesseract/a deep OCR model on
  Sagemaker or ECS/Fargate for high-volume, cost-sensitive batches — a hybrid (Textract for
  low-volume/ad hoc, self-hosted for the bulk high-volume path) is a reasonable answer.
- **Stages 2-3 (YOLO + CNN)**: batch inference on Sagemaker (Batch Transform jobs, or a fleet of
  GPU/CPU inference endpoints behind a queue) rather than always-on endpoints, to control cost —
  spin up capacity against the queue depth rather than paying for idle capacity between document
  ingestion bursts.
- **Cost/latency levers, specifically**: crop to text-line regions before running YOLO (Chapter 05)
  so each inference call is on a small image rather than a full page; batch multiple candidate crops
  per Stage 3 CNN forward pass rather than one-at-a-time; consider a cheap pre-filter (e.g., the
  contour-based size heuristic from Chapter 05) ahead of YOLO to skip pages/regions with no plausible
  candidates at all, cutting the volume of expensive model calls.
- **Observability**: track per-stage recall/precision on a sampled, periodically-reannotated
  validation set in production (Chapter 04's per-stage-vs-end-to-end distinction) so a regression in
  one stage (e.g., a new document source with unusual fonts hurting OCR recall) is caught before it
  silently degrades the end-to-end citation-tracking number.
- **Idempotency & retry**: given a queue-based architecture, each page-processing job should be
  idempotent (safe to reprocess) since at-least-once delivery is typical for SQS-style queues.

---

### 12. (System Design) How would you monitor this system in production and know when it's degrading?

**Answer:** Per Chapter 04, I'd monitor at least three signal types: (1) **per-stage proxy metrics**
on a small, continuously-refreshed labeled sample — OCR word-detection rate, YOLO candidate-recall
against known citation locations, CNN precision/recall on candidates — so a regression can be
attributed to a specific stage; (2) **distributional drift signals**, like a sudden shift in the
volume of YOLO candidates per page or in average OCR confidence scores, which can flag a new,
unfamiliar document source (e.g., a new client's slide template, or lower-quality scans) before
accuracy metrics even catch up; and (3) **downstream business metrics** — e.g., how often a human
reviewer in the MLR workflow overrides or flags a citation the pipeline reported (or missed) — as
the ground-truth signal that actually reflects the business-level "citation tracking accuracy"
number from the résumé, closing the loop between the offline eval and real-world performance.

---

### 13. (Retrospective) What would you change if you rebuilt this today?

**Answer:** A few things. First, I'd seriously evaluate collapsing Stages 2 and 3 using a modern
vision-language model (GPT-4V or similar) for at least the harder/ambiguous candidates, as discussed
above — the field has moved a lot since this was built, and a hybrid cheap-funnel-plus-VLM-adjudicator
design plausibly beats a pure YOLO+CNN stack on accuracy for a similar or lower total engineering
cost, though I'd want to benchmark cost-per-page and latency carefully before committing, since VLM
calls at millions-of-pages scale are a real cost consideration. Second, I'd invest more explicitly in
per-stage observability from day one — logging enough intermediate output (OCR boxes, YOLO
candidates pre- and post-NMS, CNN confidence per candidate) to make error attribution fast, rather
than only having the final end-to-end number to debug from. Third, I'd push harder on active
learning for the annotation pipeline — since true superscripts are rare, prioritizing human
annotation effort on the CNN's lowest-confidence or most-disagreed-upon candidates would make far
better use of a limited labeling budget than uniformly sampling documents to annotate.

---

### 14. (Retrospective) What was the hardest part of getting from 5% to 85%, technically?

**Answer:** Almost certainly the transition from "OCR text + heuristic" to "treat this as a
computer-vision detection-and-classification problem" — that's a bigger conceptual jump than any
individual model-tuning step, because it meant abandoning the assumption that the answer lived in
the text at all. Once the architecture was reframed that way, the harder day-to-day engineering work
was almost certainly the **annotation and class-imbalance problem** (Chapter 02) — getting a training
set with tight, consistent bounding boxes on a genuinely rare object class, with enough
well-chosen hard negatives (exponents, page numbers, subscripts) that the detector learned the real
discriminating signal instead of a shortcut like "flag everything small." That data-quality work is
usually the unglamorous majority of the effort behind a jump like 5% to 85%, more than any single
architectural decision.

---

### 15. (Technical) How would you evaluate whether the YOLO stage or the CNN stage is the bigger bottleneck if end-to-end accuracy plateaus?

**Answer:** I'd break the end-to-end error down using the per-stage metrics from Chapter 04: run
YOLO alone against a labeled validation set and measure recall — if YOLO is missing a meaningful
fraction of true superscripts before the CNN ever sees them, no amount of CNN improvement can fix
the end-to-end number, since those are unrecoverable false negatives (Chapter 04's error-propagation
argument). If YOLO's recall is already high but the CNN's precision/recall on the resulting
candidate set is weak, the CNN training data or architecture is the bottleneck instead. In practice
I'd build a small offline harness that runs each stage against its own held-out ground truth
independently, plus the full pipeline end-to-end, specifically so a plateau can be attributed to a
stage rather than requiring a guess-and-retrain-everything approach.

---

### 16. (System Design) This ran in production for two pharma clients, Eli Lilly and AstraZeneca — how did you keep their content and processing pipelines isolated on AWS?

**Answer:** The same pipeline code and the same trained YOLO/CNN models served both clients — I
wasn't maintaining two separate codebases — but the *data* and *access* were isolated at the
infrastructure layer, not the application layer, which is the more defensible boundary. Concretely:
each client's documents and structured citation output lived under **separate S3
prefixes/buckets**, and **IAM policies were scoped per client** so that the credentials/roles used
to process one client's batch had no read/write access to the other client's prefix — not even in
principle, not just "the application logic happens not to cross the streams." **Secrets Manager**
held per-client credentials separately rather than a single shared secret with broad access. On the
compute side, ECS Fargate tasks and Sagemaker batch/endpoint invocations were parameterized per job
by client, reading input from and writing output to only that client's S3 location, so a
misconfigured job would fail on an access-denied error rather than silently reading the wrong
client's documents. The reason this matters more than it might for a typical multi-tenant SaaS: Eli
Lilly and AstraZeneca are **competitors**, and their unpublished marketing and scientific content is
itself confidential — a cross-contamination bug here isn't just a data-hygiene embarrassment, it's a
contractual/legal problem. I'd rather over-invest in IAM-level isolation on shared infrastructure
than rely on "our code is correct" as the only thing standing between the two clients' data.

---

### 17. (System Design / Behavioral) Why did the 5%→85% accuracy improvement matter enough to justify a three-stage ML pipeline instead of a simpler heuristic, given this was going to real clients?

**Answer:** Because the cost of getting it wrong wasn't an abstract accuracy number, it was a
compliance risk sitting on top of real client content. Every citation this pipeline tracks (or
fails to track) feeds an MLR — Medical/Legal/Regulatory — review workflow for Eli Lilly and
AstraZeneca's promotional and scientific material; a claim shipped without a correctly-linked
citation, or with a citation silently dropped, is a regulatory exposure for the client, not just a
bug ticket. At 5% accuracy, the "heuristic" wasn't actually saving anyone work — a human would still
need to manually verify essentially every citation, because the system's output couldn't be
trusted, which means it was adding pipeline latency and infrastructure cost without reducing the
real bottleneck (reviewer time). That's not a viable thing to hand a paying enterprise client. The
three-stage architecture was the more complex path, but it was complex in a *targeted* way — each
stage solving a sub-problem (text/geometry extraction, candidate localization, precise
classification) that a single heuristic or a single end-to-end model couldn't solve jointly at this
accuracy bar (Chapter 04 covers exactly why). Going from 5% to 85% is what moved this from "a
demo that shows the idea works" to "a system a client's MLR team can actually build a review
workflow around," and that gap — demo vs. production-trustworthy — is exactly why the added
engineering complexity of three coordinated models was worth it here.

---

### 18. (System Design) How would you scale this on ECS Fargate + Sagemaker to handle a sudden spike in document volume from one client without affecting the other client's processing SLA?

**Answer:** The key design choice is to make capacity scale **per client, independently**, rather
than sharing one undifferentiated pool that both clients draw from — otherwise Eli Lilly uploading a
large batch ahead of a conference could starve AstraZeneca's routine daily volume, or vice versa.
Concretely: separate ECS Fargate services (or at minimum separate task definitions/queues) per
client, each with its own autoscaling policy tied to that client's queue depth in
SQS/EventBridge, so a spike in one client's queue triggers Fargate to scale that client's task count
up without touching the other client's service. On the Sagemaker side, batch transform jobs are
naturally parallelizable and don't compete for a shared always-on endpoint's capacity the way a
single real-time endpoint would; if a real-time endpoint is in play for the low-latency path, I'd
either run per-client endpoint variants or use Sagemaker's autoscaling on the endpoint with enough
headroom (and a client-aware priority/throttling layer in front of it) so one client's burst can't
exhaust the concurrency the other client depends on. I'd back this with **CloudWatch alarms on
per-client queue depth and per-client end-to-end processing latency** specifically — not just an
aggregate metric — so an SLA risk to one client is visible before it's breached, and I'd set
per-client concurrency/rate limits at the API Gateway/Lambda triggering layer as a hard backstop so
a genuine traffic spike degrades gracefully (queues longer, but bounded) rather than one client's
burst silently starving the other's throughput. The underlying principle is the same one behind the
S3/IAM isolation in the question above: shared code and shared models are fine, but capacity,
queues, and monitoring need to be scoped per client wherever a resource could otherwise become
contended between two customers who have no visibility into or control over each other's usage.
