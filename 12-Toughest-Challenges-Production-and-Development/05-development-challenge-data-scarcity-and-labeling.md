# 05 — Development Challenge: Data Scarcity & Labeling

## Why this is a development-phase problem, not a production one

Before any of the computer-vision or NLP models in this curriculum — superscript/subscript
detection (course 7), claim classification (course 9), chart/graph detection (course 10) — could go
anywhere near a Sagemaker real-time endpoint, someone had to answer a much less glamorous question:
**where does the labeled training data come from, and how do we trust it?** In a regulated pharma
domain, this is harder than it looks in a generic ML tutorial, for reasons specific to the domain
and the client relationship, not just "labeling is tedious."

## Why labeled data is scarce and expensive in this domain specifically

- **The source documents are client-confidential.** Pharma clinical, regulatory, and marketing
  documents (the kind that feed superscript/claim/chart detection models) aren't public datasets —
  there's no ImageNet-equivalent for "charts inside pharma regulatory PDFs." Every labeled example
  has to come from real or realistic client documents, under the same data-handling constraints
  covered in chapter 02, which limits how freely data can be shared even with a labeling vendor.
- **Labeling requires domain expertise, not just visual pattern-matching.** A superscript/subscript
  annotator needs to tell a footnote reference apart from a chemical formula subscript apart from a
  stray OCR artifact; a claim-classification annotator needs to know what counts as a "claim"
  requiring MLR (medical/legal/regulatory) substantiation versus general descriptive text. Generic
  crowd-labeling is a poor fit; the annotators who can do this reliably are expensive and scarce.
- **Class imbalance is severe and domain-driven.** Most tokens in a document aren't superscripts;
  most chart images aren't the rare-but-important chart type (e.g., a forest plot versus a generic
  bar chart); most sentences in a clinical document aren't claims requiring the strictest review
  tier. A naively collected dataset systematically under-represents exactly the cases that matter
  most to get right.

## The core techniques used to make scarce, expensive labels go further

### Annotation quality and consensus

A single annotator's label on an ambiguous case (is this superscript a footnote marker or a
chemical formula? is this sentence a "claim" or just descriptive background?) isn't reliable enough
to train against directly. The standard fix is **multi-annotator overlap with a consensus/adjudication
step**: a meaningful fraction of examples get labeled by two or more annotators independently, and
disagreements get resolved by a senior/domain-expert adjudicator, with the disagreement rate itself
tracked as a signal — a class with a high inter-annotator disagreement rate usually means the label
schema itself needs clarification, not just "more careful annotators." AWS Ground Truth (used for
chart/graph detection labeling in course 10) supports exactly this pattern natively — multiple
worker responses per object with consensus/annotation-consolidation built in.

### Active learning to prioritize what gets labeled

Because expert annotation time is the scarce resource, labeling budget should go to the examples
that will most improve the model, not to examples chosen at random. A practical active-learning loop:
train an initial model on the labeled data available so far, run it over a large pool of unlabeled
documents, and prioritize human labeling for the examples where the model is **least confident**
(closest to its decision boundary) or **most likely to be a rare/underrepresented class**. This
turns a fixed annotation budget into a much larger effective dataset than random sampling would,
and is especially valuable for the severe class-imbalance problem above — it deliberately surfaces
more of the rare, high-value examples for labeling rather than mostly re-confirming the common case.

### Synthetic data augmentation

For the computer-vision tasks specifically (superscript detection, chart detection), synthetic
augmentation can multiply a scarce labeled set considerably: programmatically generating documents
with known superscript/subscript placements and known ground-truth bounding boxes (varying font,
DPI, scan noise, rotation to mimic real scanned-document artifacts), or generating synthetic chart
images with known chart-type labels and known axis/legend positions. Synthetic data is never a full
substitute for real labeled examples — models trained purely on synthetic data tend to underperform
on the visual noise and edge cases real scanned client documents actually have — but it's a strong
complement: pretrain or heavily augment on synthetic data, then fine-tune on the smaller pool of
real, expert-labeled examples.

## Illustrative struggle: the model that looked fine until the imbalance showed up

> **Illustrative, plausible scenario — not a verified account of a specific project's development
> history.** Built from the real class-imbalance and scarce-labeling dynamics described above.

**Situation.** During development of a superscript/subscript detection model (course 7) for a
pharma client's document-processing pipeline, an early model trained on the first batch of
expert-labeled documents scored well on held-out accuracy — but accuracy was dominated by the
overwhelming majority of "no superscript here" regions, and the metric was masking poor recall on
the actual superscript/subscript instances the client cared about, which were a small fraction of
the total labeled area.

**Task.** I needed to get real visibility into per-class performance (not just aggregate accuracy),
figure out whether the problem was model capacity or data scarcity on the minority class, and fix
it within a labeling budget that was already tight because expert annotator time was the limiting
resource, not compute.

**Action.** I switched the primary evaluation metric to per-class precision/recall/IOU rather than
aggregate accuracy, which confirmed the model was substantially under-recalling true superscripts,
especially small chemical-formula subscripts against noisy scanned backgrounds. Rather than
requesting a much larger general labeling batch, I ran an active-learning pass: scored a large pool
of unlabeled pages with the current model, and prioritized expert annotation for the pages where the
model's confidence was lowest specifically in superscript/subscript regions, which concentrated the
scarce annotation budget on exactly the underrepresented, hard cases. I paired that with a synthetic
augmentation pass — programmatically generating scanned-document-style images with known superscript
placements and realistic noise — to give the model more exposure to the minority class without
consuming any further expert annotation budget. I also introduced a two-annotator consensus step
for the newly prioritized hard cases, since these were exactly the ambiguous examples most likely to
have inconsistent labels.

**Result.** *(Illustrative)* Recall on the superscript/subscript minority class improved
substantially (illustrative: roughly 15–20 percentage points) without a proportional increase in
total annotation hours, because the active-learning prioritization concentrated expert time on the
cases that actually moved the metric. The per-class evaluation dashboard and the active-learning
prioritization step became standard practice for the subsequent claim-classification (course 9) and
chart-detection (course 10) labeling efforts.

## Talking points this chapter sets up

- Why aggregate accuracy is a misleading metric under severe class imbalance, and what to use
  instead.
- Active learning as a way to make an expensive, domain-expert-limited labeling budget go further,
  by targeting the model's actual uncertainty rather than random sampling.
- Multi-annotator consensus as both a quality control and a diagnostic — high disagreement rate on
  a class often means the label schema is ambiguous, not that annotators are careless.
- Synthetic augmentation as a complement to, not a replacement for, scarce real expert-labeled data.
