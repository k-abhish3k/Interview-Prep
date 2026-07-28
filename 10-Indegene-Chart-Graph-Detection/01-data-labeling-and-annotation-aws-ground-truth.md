# 01 — Data Labeling & Annotation with AWS Ground Truth

## Why this chapter matters

"Fine-tuned YOLOv5" sounds like the interesting part of this project, but it's downstream of a much
less glamorous, equally decisive step: **somebody had to draw thousands of correct bounding boxes
around charts and graphs before any model could be trained.** Interviewers who work in applied ML
know this, and a resume line that says "performed collaborative labeling on AWS Ground Truth" is an
invitation to probe whether you actually understand annotation as an engineering and quality problem
— not just "we hired some labelers." This chapter builds that vocabulary from first principles, then
walks through AWS Ground Truth specifically. Worth keeping in mind throughout: the source documents
being labeled here were real Eli Lilly and AstraZeneca content, feeding a model that went on to run in
production for both clients, not a generic public dataset — which is part of why label consistency and
quality process mattered as much as they did.

## Bounding Boxes: The Annotation Format

For object detection, the label for one object instance is a **bounding box** — the smallest
axis-aligned rectangle that fully contains the object — plus a class label. There are two common
encodings you'll see across tools and papers:

- **Corner format**: `(x_min, y_min, x_max, y_max)` — top-left and bottom-right pixel coordinates.
  This is what most annotation UIs, including Ground Truth, store internally.
- **Center format (YOLO)**: `(x_center, y_center, width, height)`, all normalized to `[0, 1]` by
  dividing by image width/height. YOLOv5 trains on this format specifically, so any Ground Truth
  export needs a conversion step (Chapter 02 shows the resulting `data.yaml` and label-file layout).

For chart/graph detection, each labeled document image can contain zero, one, or several
chart/graph instances — a slide might show one large chart, a clinical report page might show a
forest plot next to a small inset bar chart. So a single image's label file isn't one box, it's a
list of boxes, each with its own class (in this project realistically a single class, `chart`, or a
small set like `bar_chart` / `line_chart` / `table` if the taxonomy was split more finely).

## What Makes a "Good" Bounding Box

A box that's too loose (extra whitespace/margin around the chart) teaches the model to be imprecise
about localization; a box that's too tight (clipping the axis labels or legend) teaches the model to
under-predict extent. The practical convention — and the one worth stating in an interview — is:
**the box should tightly enclose the full visual extent of the chart, including its axis labels,
legend, and title if those are visually part of the chart region**, with no more than a couple of
pixels of margin. This convention matters enormously downstream, because the IOU-based evaluation
metric (Chapter 03) directly penalizes boxes that don't match this convention consistently — a model
trained on inconsistently-drawn ground truth can't outperform the consistency of its labels.

## Annotation Quality: Consensus Across Multiple Labelers

"Collaborative labeling" in the resume bullet is the key phrase here — this wasn't one person
labeling everything, it was a **team**, which introduces the central quality problem of any
supervised-learning data pipeline: different humans draw boxes slightly differently, and some
mislabel outright.

The standard mitigations, roughly in order of cost:

1. **Clear labeling instructions with visual examples.** The single highest-leverage intervention:
   an instruction doc showing correct vs. incorrect examples (e.g., "include the legend," "don't
   include surrounding whitespace," "a table is not a chart") removes most disagreement before it
   happens.
2. **Multi-annotator overlap + agreement measurement.** Have more than one labeler annotate a subset
   of the same images, then measure agreement — for bounding boxes, this is typically done with IOU
   between the two labelers' boxes on the same object (see `notebooks/01_bounding_box_annotation_demo.ipynb`
   for a hands-on IOU-based inter-annotator agreement calculation). A low average IOU between
   labelers on the same images signals ambiguous instructions or an ambiguous class definition, not
   just "one labeler is bad."
3. **Consensus/adjudication.** When labelers disagree beyond some threshold, a senior
   reviewer/adjudicator resolves the conflict — either by picking one annotation, drawing a corrected
   box, or removing the ambiguous instance from the training set entirely if it's genuinely
   unclear (e.g., a chart embedded inside a photo, a very small inset chart).
4. **Spot-check sampling.** Rather than reviewing every single annotation (expensive at scale), a
   random or risk-weighted sample is reviewed for quality, with corrective feedback looped back to
   labelers whose error rate is elevated.

This is exactly the kind of process detail worth having ready for an interview follow-up like "how
did you handle labelers disagreeing?" — see `99-Interview-QA.md` for a full worked answer.

## Active Learning: Prioritizing What to Label Next

Labeling every candidate image is wasteful — most document pages are dominated by easy, redundant
examples (a lot of body-text-only slides, a lot of near-duplicate chart styles from the same
template deck). **Active learning** is the practice of using a partially-trained model to decide what
to label next, rather than labeling in a random or arbitrary order:

1. Train an initial model on a small seed set of labeled images.
2. Run that model over the large pool of *unlabeled* images.
3. Prioritize for human labeling the images where the model is **least confident** (predictions near
   the decision boundary, or images where it flags a region with low confidence that a human should
   verify), or where it disagrees most with itself under augmentation (e.g., slight crops/flips
   producing inconsistent boxes).
4. Label that prioritized batch, retrain, repeat.

For chart/graph detection specifically, this matters because pharma slide decks are highly
templated — most charts in a given client's deck library look visually similar (same color scheme,
same axis style), so a model converges quickly on the common case. Active learning steers scarce
labeling budget toward the *unusual* cases — an oddly formatted forest plot, a chart with an
unconventional legend placement, a chart partially overlapping a text box — which is exactly where a
production model is most likely to fail without targeted examples.

## AWS Ground Truth: How the Managed Service Works

AWS Ground Truth is Sagemaker's managed data-labeling service. It exists to handle the workforce
management, UI, and quality-control scaffolding around annotation, so a team doesn't have to build a
labeling tool from scratch. Three concepts map directly onto how this project would have used it:

### Labeling Workforces

Ground Truth lets you route labeling jobs to one of three workforce types:

- **Private workforce** — your own team members (internal Indegene staff, plausibly including
  the candidate and teammates doing "collaborative labeling") log in and label through a
  Ground-Truth-hosted UI. This is the natural choice when the content is client-confidential pharma
  material that can't go to an anonymous public workforce.
- **Amazon Mechanical Turk** — a public, on-demand crowd workforce. Fast and cheap for
  non-sensitive, generic labeling tasks, but not appropriate for content under pharma client
  confidentiality.
- **Vendor workforce** — a managed third-party labeling company procured through AWS Marketplace.

Given the confidentiality profile of pharma marketing/regulatory content, a **private workforce** is
the realistic choice for this project.

### Job Configuration

A Ground Truth labeling job is configured with: the input data location (an S3 manifest listing
image object keys), the task type (built-in **bounding box** task type for object detection, which
provides the draw-a-rectangle UI out of the box), the labeling instructions and category list shown
to annotators, and — importantly — an optional **automated data labeling** setting, where Ground
Truth trains an internal model as labeling progresses and starts auto-labeling images it's confident
about, sending only the uncertain ones to humans (this is active learning as a built-in product
feature, conceptually identical to the manual process described above).

### Output Manifest Format

Ground Truth writes labeled results as an **augmented manifest** — a JSON-lines file where each line
corresponds to one input image and contains the original data reference plus the annotation output,
roughly:

```json
{
  "source-ref": "s3://indegene-chart-dataset/slides/deck042_page07.png",
  "bounding-box": {
    "image_size": [{"width": 1920, "height": 1080, "depth": 3}],
    "annotations": [
      {"class_id": 0, "top": 210, "left": 340, "width": 560, "height": 410}
    ]
  },
  "bounding-box-metadata": {
    "objects": [{"confidence": 0.94}],
    "class-map": {"0": "chart"},
    "type": "groundtruth/object-detection",
    "human-annotated": "yes"
  }
}
```

Note the box is stored in `(top, left, width, height)` pixel format — this needs converting to
YOLOv5's normalized center-format `.txt` label files before training, which Chapter 02 covers as
part of the fine-tuning data pipeline.

## Tying It Back

Every concept in this chapter is the "before" half of the resume bullet. "Performed collaborative
labeling on AWS Ground Truth to annotate the target objects" is shorthand for: defining a bounding-box
convention, coordinating multiple labelers through a private Ground Truth workforce, managing
annotation quality and disagreement, and (plausibly) using active-learning-style prioritization to
get the most value out of limited labeling time on a highly templated but occasionally unusual
document corpus. This is the vocabulary to reach for when an interviewer asks "walk me through how
you built the training dataset" — not "we used Ground Truth," but *why* each of those decisions
mattered for getting a 96%-TPR model out the other end.
