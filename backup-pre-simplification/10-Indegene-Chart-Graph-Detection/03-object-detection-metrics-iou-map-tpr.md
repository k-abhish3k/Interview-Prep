# 03 — Object Detection Metrics: IOU, mAP, and TPR

## Why this chapter matters

"Attained a TPR of 96% with an IOU threshold set at 0.85" is the single most important sentence on
this resume line — it's the number an interviewer will anchor on, and it's also the sentence most
likely to draw a follow-up like "why 0.85 instead of the usual 0.5?" or "what's the difference
between that and mAP?" If you can't explain *precisely* what that sentence means, the achievement
reads as a memorized statistic instead of an understood result. This chapter derives every term in it
from scratch. It's also worth remembering this wasn't measured against an academic benchmark for its
own sake — this model ran in production for Eli Lilly and AstraZeneca, so every point of TPR below
100% represented real charts that would have silently skipped their downstream processing step on
actual client documents (see `00-README.md` and Q17 in `99-Interview-QA.md`).

## Intersection over Union (IOU)

IOU is the fundamental measure of how well two bounding boxes overlap. Given a predicted box and a
ground-truth box, IOU is:

```
IOU = area(prediction ∩ ground_truth) / area(prediction ∪ ground_truth)
```

- IOU = 1.0 means the boxes are identical.
- IOU = 0.0 means the boxes don't overlap at all.
- IOU near 0.5 means roughly half the combined area is shared overlap — a rough, "in the right place"
  match.
- IOU near 0.85+ means the boxes are nearly coincident — a tight, precise match.

In code, for axis-aligned boxes `(x_min, y_min, x_max, y_max)`:

```python
def iou(box_a, box_b):
    xa1, ya1, xa2, ya2 = box_a
    xb1, yb1, xb2, yb2 = box_b

    inter_x1, inter_y1 = max(xa1, xb1), max(ya1, yb1)
    inter_x2, inter_y2 = min(xa2, xb2), min(ya2, yb2)
    inter_area = max(0, inter_x2 - inter_x1) * max(0, inter_y2 - inter_y1)

    area_a = (xa2 - xa1) * (ya2 - ya1)
    area_b = (xb2 - xb1) * (yb2 - yb1)
    union_area = area_a + area_b - inter_area

    return inter_area / union_area if union_area > 0 else 0.0
```

`notebooks/03_iou_map_metrics_from_scratch.ipynb` implements exactly this and builds the rest of the
metrics in this chapter on top of it, in pure numpy.

## The IOU Threshold: Defining a "Correct" Detection

Raw IOU is a continuous number; evaluating a detector requires a binary decision per prediction:
"does this count as a correct detection or not?" That binary decision is made by picking an **IOU
threshold** and applying the rule: *a predicted box counts as a True Positive (TP) if it has IOU ≥
threshold with a ground-truth box of the same class (and that ground-truth box hasn't already been
matched to a higher-confidence prediction); otherwise it's a False Positive (FP).* Any ground-truth
box left unmatched is a False Negative (FN) — a real chart the model missed entirely, or found but
with too little box overlap to count.

**This threshold is a business decision baked into the evaluation, not a fixed law of nature.** The
most common convention in general object-detection benchmarks (COCO, Pascal VOC) is **IOU = 0.5** —
a prediction only needs to overlap the ground truth by half to be counted correct. That's a
deliberately forgiving bar, appropriate for tasks like "did the model notice there's a car roughly
here," where rough localization is good enough.

**IOU = 0.85 is a much stricter bar.** At that threshold, a prediction has to be a *near-exact* fit to
the true chart region — off by a modest margin on any edge (missing part of the legend, including
extra whitespace, clipping an axis label) drops the IOU below 0.85 and the detection is scored as
wrong, even though a human glancing at the same box would call it "basically right." Why does that
matter for this project specifically? Because the downstream consumer of a chart detection isn't a
human eyeballing the result — it's very likely another automated pipeline stage that **crops the
image at the predicted box** to hand off to chart-specific processing (digitization, redaction,
separate MLR review). A loosely-fit box at IOU 0.5 might crop off part of the chart's legend or
include a chunk of adjacent body text in the crop, corrupting whatever the next stage does with it.
Evaluating at IOU 0.85 is a direct proxy for "is this box tight enough to safely crop on," which is
the real production requirement — not just "did the model notice a chart was there."

## True Positive Rate (TPR) / Recall

TPR — also called **recall** — answers: *of all the actual chart/graph instances in the evaluation
set, what fraction did the model successfully detect (at the chosen IOU threshold)?*

```
TPR (Recall) = TP / (TP + FN) = TP / (total actual positives)
```

**"96% TPR at IOU 0.85"** means: across the held-out evaluation images, the model correctly and
tightly (IOU ≥ 0.85) detected 96% of the actual chart/graph regions present. Only 4% of real charts
were either missed entirely or detected with a box too loose to count as a match at that strict
threshold.

It's worth being explicit about what this metric does *not* tell you: TPR alone says nothing about
**false positives** — a model that (hypothetically) predicted a chart box on every single page,
whether or not one was really there, could still score close to 100% TPR, because TPR only counts
what happens to the real charts, not how many spurious boxes came along with them. That's exactly why
TPR is normally reported alongside **precision** (see below), and why an interviewer might reasonably
ask "what was your precision / false positive rate?" as a natural follow-up — a good answer
acknowledges that TPR is one half of the picture, and that a production system would also track
precision (and probably a business-relevant proxy for it, e.g. "how many detections per page needed
manual review").

## Precision and the Precision-Recall Curve

**Precision** answers the complementary question: *of all the boxes the model predicted, what
fraction were actually correct?*

```
Precision = TP / (TP + FP)
```

Precision and recall trade off against each other as you sweep the model's confidence threshold:
lowering the confidence threshold reports more boxes (catching more true positives, raising recall)
but also more false positives (lowering precision); raising it does the opposite. Plotting precision
against recall as this threshold sweeps produces the **precision-recall (PR) curve**, and the
area under that curve for one class is the **Average Precision (AP)** for that class at the chosen
IOU threshold.

## mAP (mean Average Precision)

**mAP** is the standard single-number summary metric for object detection: compute AP per class
(area under that class's PR curve), then average across all classes. "Mean" refers to averaging
across classes — with a single `chart` class, as this project plausibly used, mAP for that IOU
threshold collapses to that one class's AP. You'll also see **mAP@0.5** (AP at IOU=0.5) and
**mAP@0.5:0.95** (COCO's stricter convention: average AP across IOU thresholds from 0.5 to 0.95 in
0.05 steps) reported as standard YOLOv5 training/validation output — worth recognizing those labels
directly in YOLOv5's console output and `results.csv` during training.

The resume bullet reports **TPR at a single, strict, fixed IOU threshold (0.85)** rather than mAP —
that's a reasonable choice when the business question is specifically "how often does the model find
the real chart with a tight-enough box to crop on," which is a recall-first framing suited to a
triage/routing use case, versus mAP's broader "how good is the whole precision-recall tradeoff"
framing suited to comparing detector architectures in a benchmark setting.

## Putting the Number in Context

| Threshold | What it demands | 96% TPR at this threshold means... |
|---|---|---|
| IOU ≥ 0.5 (common default) | Roughly half the box area overlaps | A much lower bar — most models score higher here than at 0.85 |
| IOU ≥ 0.85 (this project) | Near-exact box match | The model isn't just noticing charts, it's localizing them precisely enough for automated cropping |

`notebooks/03_iou_map_metrics_from_scratch.ipynb` builds a small synthetic set of predicted vs.
ground-truth boxes and computes TP/FP/FN, TPR, precision, and a simplified per-class AP at IOU=0.85 —
reproducing the exact shape of the resume's headline computation on toy data, entirely offline.

## Tying It Back

When an interviewer asks "walk me through what 96% TPR at IOU 0.85 actually means," the answer is:
IOU measures box overlap; 0.85 is a strict definition of "correct" that requires near-exact
localization, not just rough presence detection; TPR/recall measures what fraction of real charts
were found under that strict definition; and choosing that strict threshold — rather than the more
common 0.5 — reflects that the model's output feeds an automated cropping step downstream, where a
loosely-fit box would corrupt the next pipeline stage. That's the difference between reciting a
number and demonstrating you understand what it cost to earn it.
