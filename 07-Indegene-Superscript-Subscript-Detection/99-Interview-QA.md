# 99 — Interview Q&A

Read this last, once Chapters 00–07 are fresh. Questions are grouped behavioral -> technical ->
system design -> retrospective, roughly the order a real interview tends to escalate, with one
"gotcha" question (Q2) positioned early and deliberately — it's the question most likely to expose
whether a candidate has actually thought through this pipeline's lifecycle or is only describing its
happy path.

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

### 2. (Gotcha) If a corrected re-scan of a document comes back through the pipeline, how do you avoid double-counting its citations?

**Answer:** Honestly — as built, this pipeline doesn't avoid that today. Every document that lands
in S3 triggers a brand-new run of the full OCR-YOLO-CNN pipeline, and there's no check anywhere in
that flow for "have I already produced a structured citation output for this document before." A
resubmission — say, an MLR reviewer asks for a page to be rescanned because it was too blurry for
OCR to read a footnote reliably — produces a **second, independent set of citation detections**
sitting alongside the first, with nothing linking them. The workaround that exists today is entirely
manual and operator-driven: someone finds the prior output for that document, marks it stale in
whatever system tracks MLR review status, and lets the resubmission process fresh. That works, but
only if someone remembers to do it every time.

It's also worth being precise that there are two genuinely different things hiding inside "the
document came in again," and conflating them is a mistake: **a genuinely new document** — different
content that's never been processed — where fresh processing from scratch is exactly correct
behavior, versus **a corrected re-scan of something already processed** — same underlying claims and
citations, different input bytes (higher-DPI rescan, recompressed export, a deskew fix) — where fresh
processing isn't *wrong*, it's just redundant and produces an unwanted duplicate. The pipeline can't
currently tell these apart at all; both look identical to Stage 1, which just sees "a new image
arrived."

If I were building a real fix, I'd add a cheap dedup check ahead of the expensive three-stage
pipeline: an exact SHA-256 hash of the input bytes to catch true duplicate re-uploads outright (skip
reprocessing, return a pointer to the existing output), and a perceptual hash (a difference hash over
the decoded image, compared by Hamming distance) tolerant of the kind of change a corrected re-scan
actually introduces, to flag likely revisions as a "probably supersedes an existing job" signal —
with a human confirming the supersede relationship before anything gets marked stale, since a
similarity threshold is a heuristic, not a certainty, and I'd rather over-process a document than
silently drop or merge one that turned out to be genuinely different content. I'd tag every citation
record with a `source_document_version` field — which input hash and which model version produced
it — so "this citation's source document changed" and "this citation's scoring model changed" (a
related but distinct concern, see Q19 below) are both attributable, not conflated. And I'd want this
to reach past this pipeline's own database: if there's an MLR reporting layer or a citation-search
surface downstream — and there almost certainly is — a duplicated detection set there means
double-counted compliance metrics or a search result surfacing stale, superseded citation content.
The full reasoning behind this is in `06-document-reprocessing-and-citation-deduplication.md`, and
`notebooks/05_content_hash_reprocessing_demo.ipynb` runs the exact-hash/perceptual-hash/supersede
logic end to end against synthetic images.

---

### 3. (Behavioral) Why did you choose a multi-stage pipeline instead of one end-to-end model?

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

### 4. (Technical) Why not just use OCR font-size heuristics instead of a full detection+classification pipeline?

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

### 5. (Technical) How do you handle class imbalance when superscripts are a tiny fraction of all characters?

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

### 6. (Technical) Walk me through IOU and non-max suppression — why does YOLO need NMS at all?

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

### 7. (Technical) YOLOv5 vs. newer YOLO versions (v8, v10, etc.) — what are the trade-offs?

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

### 8. (Technical) Why add a CNN classifier after YOLO instead of just improving YOLO's own classification head or raising its confidence threshold?

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

### 9. (Technical) How would GPT-4 Vision (or a general vision-language model) change this pipeline if you rebuilt it today?

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

### 10. (Technical) How is Sagemaker used in a pipeline like this?

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

### 11. (Technical) What's the difference between text detection and text recognition in OCR, and why does that distinction matter here?

**Answer:** Detection finds *where* text is (bounding boxes/regions); recognition decodes *what* the
text says, given a region known to contain it. The distinction matters here because Stage 1's
detection output — word/line bounding boxes and baseline metadata — is the geometric scaffold Stage
2's YOLO model operates on; the actual recognized text content matters less to Stage 2 than the
geometry does. If I only had OCR's recognized text string with no positional metadata, I'd have
nothing for YOLO to search near, and the whole "look for a small, elevated region relative to this
line's baseline" strategy wouldn't be possible — which is exactly why Chapter 01 emphasizes that
OCR's contribution to this pipeline is geometry as much as it is text.

---

### 12. (System Design) Design this to run at scale over millions of pages with acceptable latency and cost.

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
  idempotent (safe to reprocess) since at-least-once delivery is typical for SQS-style queues — see
  Q18 below for the concrete concurrency risk this addresses and Chapter 06/07 for the proposed
  content-hash-based fix.

---

### 13. (System Design) How would you monitor this system in production and know when it's degrading?

**Answer:** Per Chapter 04, I'd monitor at least three signal types: (1) **per-stage proxy metrics**
on a small, continuously-refreshed labeled sample — OCR word-detection rate, YOLO candidate-recall
against known citation locations, CNN precision/recall on candidates — so a regression can be
attributed to a specific stage; (2) **distributional drift signals**, like a sudden shift in the
volume of YOLO candidates per page or in average OCR confidence scores, which can flag a new,
unfamiliar document source (e.g., a new client's slide template, or lower-quality scans) before
accuracy metrics even catch up — this is exactly the signal that would have surfaced Chapter 07's
"confidence threshold left at a debug value" bug much sooner than a human noticing the MLR review
queue had gotten unusually large; and (3) **downstream business metrics** — e.g., how often a human
reviewer in the MLR workflow overrides or flags a citation the pipeline reported (or missed) — as
the ground-truth signal that actually reflects the business-level "citation tracking accuracy"
number from the résumé, closing the loop between the offline eval and real-world performance.

---

### 14. (Retrospective) What would you change if you rebuilt this today?

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
better use of a limited labeling budget than uniformly sampling documents to annotate. Fourth, I'd
build the content-hash dedup check and the `source_document_version`/model-version tagging from
Chapter 06/07 in from day one rather than as a retrofit — both are the kind of thing that's cheap to
design in up front and genuinely annoying to bolt onto a system that's already processed months of
untagged, unlinked output.

---

### 15. (Retrospective) What was the hardest part of getting from 5% to 85%, technically?

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

### 16. (Technical) How would you evaluate whether the YOLO stage or the CNN stage is the bigger bottleneck if end-to-end accuracy plateaus?

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

### 17. (System Design) This ran in production for two pharma clients, Eli Lilly and AstraZeneca — how did you keep their content and processing pipelines isolated on AWS?

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

### 18. (System Design / Behavioral) Why did the 5%→85% accuracy improvement matter enough to justify a three-stage ML pipeline instead of a simpler heuristic, given this was going to real clients?

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

### 19. (System Design) How would you scale this on ECS Fargate + Sagemaker to handle a sudden spike in document volume from one client without affecting the other client's processing SLA?

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

---

### 20. (Technical, follow-up to Q2) Under an at-least-once queue like SQS/EventBridge, could the same document actually get processed twice by two different workers — and would you even notice?

**Answer:** Yes, and this is worth being direct about rather than assuming a batch system is
idempotent just because it's queue-based. If a worker task is evicted mid-processing (a Fargate spot
interruption, a rolling deploy) without cleanly acknowledging its queue message, the message's
visibility timeout expires and a second worker can pick up and process the exact same document —
both workers unaware of each other, potentially both completing and both writing a structured
citation output for the same document. Whether you'd *notice* depends entirely on whether anything
downstream is checking for duplicate output per document — and per Chapter 07, absent a shared
idempotency check at job pickup, nothing does today. The practical fix is the same content-hash check
proposed in Chapter 06 for resubmitted documents: if every worker checks a shared processed-document
index before running the expensive pipeline stages, a duplicate delivery resolves to a cheap no-op.
Short of that, a lightweight distributed claim-check per `document_id` (first worker to claim a row
in a shared table wins; a second delivery is a no-op) is the narrower fix. The honest answer is that
today this system is not *provably* idempotent under retry — it's a real, named gap, not a solved
problem, and `07-production-resilience-and-operational-engineering.md`'s concurrency section covers
the reasoning in full.

---

### 21. (Technical, follow-up to Q2) A retrained CNN gets redeployed to the same Sagemaker endpoint — what stops a batch of old-model and new-model results from getting silently compared as if they came from the same system?

**Answer:** Nothing, unless the deployment pipeline and the output schema both enforce it — which is
exactly the bug Chapter 07 walks through: a retrained model redeployed under the same endpoint
name/alias without a version identifier means every citation record produced afterward is
indistinguishable, in the data, from one produced by the old model. If an accuracy sample audit then
shows a shift, there's no way to attribute it to the model change versus a new document source or
sampling noise, because nothing recorded says which model produced which detection. The fix is
structural, not procedural: make the model-version fields inside `source_document_version` (Chapter
06) a required, non-nullable part of the output schema, and make the deployment pipeline itself
reject any model push that doesn't carry a version identifier — so it's impossible to produce an
untraceable citation record, rather than relying on a human to remember to tag every deploy. This is
the model-version half of the same underlying problem Q2 covers for the document-version half: a
citation record has to be traceable to *both* the source document version and the model version that
produced it, and conflating "which axis changed" is exactly the mistake to avoid.

---

### 22. (System Design, follow-up to Q2) If you found the false-positive-flood bug from a debug confidence threshold in production, how would you have caught it before a human noticed the MLR review queue backing up?

**Answer:** By monitoring candidate volume per page as a distributional signal, not waiting on
accuracy metrics or human complaints to surface it (Chapter 07, bug #2, and Q13 above). A confidence
threshold left at a debug value doesn't crash anything — the pipeline keeps running and keeps
producing plausible-looking output, just many times more Stage 2 candidates than expected, each one
costing a Stage 3 Sagemaker call. A simple CloudWatch alarm on "candidates-per-page relative to a
trailing historical baseline" would have flagged a multi-x jump within the first batch it ran on,
long before it showed up as either a cost spike or a human reviewer noticing more flagged citations
than usual. More broadly, this is why Chapter 07 argues for required, environment-scoped
configuration rather than defaults that "happen to" be safe — a debug threshold should never be able
to reach production silently in the first place, which is a stronger fix than only catching it after
deployment via monitoring.
