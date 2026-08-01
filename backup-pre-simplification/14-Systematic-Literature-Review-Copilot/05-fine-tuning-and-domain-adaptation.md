# 05 — Fine-Tuning and Domain Adaptation

## What needs adapting, specifically

Chapter 2's Reason 2 named the problem at a high level: screening reliability depends on the model
correctly applying a client's therapeutic-area vocabulary and the specific shape of a given review's
inclusion/exclusion criteria. This chapter is the "how" — the concrete mechanism for closing that gap,
and the evaluation discipline for knowing whether it actually worked.

Two distinct things benefit from adaptation, worth separating because they call for slightly
different training data:

1. **Therapeutic-area vocabulary.** An oncology review's criteria reference tumor staging systems,
   response-evaluation criteria (RECIST-style endpoints), specific biomarker terminology; a
   cardiology review's criteria reference ejection fraction thresholds, MACE (major adverse
   cardiac event) composite endpoints, specific trial-design terms for that field. A base model's
   general medical knowledge from pretraining is a reasonable starting point, but reliable
   application of a *specific* therapeutic area's terminology, at the precision a screening decision
   needs, benefits from adaptation on examples drawn from that area specifically.
2. **Inclusion/exclusion criteria reasoning patterns.** Beyond vocabulary, different reviews' criteria
   have different *structures* — some are narrowly numeric (age range, dose threshold, follow-up
   duration), some are more interpretive (study design quality, outcome relevance) — and a model
   needs to reliably map an abstract's content onto whatever criteria structure a given review
   actually uses, not just recognize domain terms in isolation.

## LoRA/QLoRA, applied here rather than re-derived

Course 11's chapter 02 covers the LoRA mechanism in full mathematical depth — a frozen pretrained
weight matrix `W`, two small trainable low-rank matrices `A` and `B` such that the effective weight
becomes `W + B·A`, with only `A` and `B` updated during training, keeping the trainable parameter
count a small fraction of a full fine-tune. QLoRA extends the same idea with a quantized frozen base
model, further cutting the memory needed to hold the frozen weights during training. This project
uses the identical mechanism course 11 describes — it isn't re-derived here — applied to a different
target: not clinical-document generation format and register (course 11's ICF/PLPS/SOC use case), but
**screening-decision and extraction reasoning conditioned on therapeutic-area vocabulary and a
review's specific criteria patterns**.

The practical shape this takes: a base DeepSeek model, with **separate LoRA adapters trained per
therapeutic area** (an oncology adapter, a cardiology adapter, and so on) rather than one adapter
trying to cover every domain at once — mirroring course 11's own "one frozen base, multiple
swappable adapters" serving pattern (course 11's chapter 03), which SageMaker's multi-adapter serving
support makes operationally practical: one deployed base model, adapters swapped or served
concurrently depending on which therapeutic area's reviews are actively screening. Training data for
each adapter draws from that therapeutic area's historical, human-adjudicated screening decisions —
abstracts a human reviewer already screened against a real set of inclusion/exclusion criteria, with
the human's actual include/exclude verdict and stated reasoning as the target the adapter learns to
reproduce.

## Why not prompt engineering alone, and why not full fine-tuning

Applying course 11's chapter 02 framing (a ladder of increasing cost and increasing behavioral
control) to this project's specific need: prompt engineering alone — putting the criteria text and a
few worked examples directly in the prompt — is the cheapest option and a reasonable starting point,
but its reliability degrades as criteria get more nuanced and as the volume of repeated calls grows
into the thousands; "usually applies the criteria correctly when reminded" is not the bar a
regulatory-submission-supporting screening decision needs to clear. Full fine-tuning (updating every
weight) is the most expensive option and generally overkill here too — the model's general medical
and scientific reading comprehension from pretraining is already a strong foundation; what needs
changing is a comparatively narrow, behavioral thing (reliable, consistent application of a specific
vocabulary and criteria structure), which is precisely the case LoRA is designed for: adapt behavior
without disturbing the broad general competence the frozen base weights already encode.

## Evaluating fine-tuned screening quality: precision and recall against a gold-standard set

The standard, real SLR quality metric worth naming directly, because "the model seems to be doing
well" is not an answer that survives a technical follow-up: build a **gold-standard evaluation set** —
a subset of abstracts a qualified human reviewer (or, per common SLR double-screening practice, two
independent reviewers with disagreements adjudicated by a third) has already screened definitively —
and measure the fine-tuned model's screening decisions against those human verdicts using standard
**precision and recall**:

- **Screening recall** (sensitivity) — of the abstracts a human reviewer would have included, what
  fraction did the model also include? This is the metric that matters most in SLR practice, because
  the cost of a **false exclusion** (the model wrongly screens out a paper that should have been
  included) is asymmetric and severe: an excluded paper never reaches full-text review or extraction
  at all, so a missed inclusion can mean a genuinely relevant study silently never makes it into the
  review's evidence base — exactly the kind of gap a regulator or auditor could later find.
- **Screening precision** — of the abstracts the model included, what fraction would a human reviewer
  agree with? Lower precision is a real cost (more false-positive inclusions mean more downstream
  full-text/extraction reviewer effort spent on papers that ultimately get excluded on closer read),
  but it's a recoverable cost — a human still reviews everything the model includes downstream. A
  false exclusion, by contrast, typically never gets a second look at all unless something else
  (an audit, a spot-check sample) catches it.

That asymmetry is worth stating explicitly if asked how you'd tune the model's decision threshold:
**bias the operating point toward higher recall, accepting somewhat lower precision**, because the
downstream human-review gate (Chapter 6, Chapter 8) is specifically positioned to catch and correct a
false-positive inclusion, while nothing downstream of screening is positioned to catch a false-
negative exclusion unless a separate spot-check process is deliberately built for it. A reasonable,
defensible practice worth naming: periodically spot-check a sample of the model's *excluded* abstracts
against human judgment too, precisely because that's the failure mode with no natural downstream
catch.

## The retraining loop as therapeutic-area focus shifts

Chapter 7 covers model-version and criteria-version staleness as tracked, independent axes in depth;
the fine-tuning-specific piece of that story belongs here. As a client's therapeutic-area focus shifts
— a new indication added, an existing one's typical criteria patterns evolving as the client's own
review methodology matures — the LoRA adapters trained on historical screening decisions for that
area gradually become less representative of current practice. The retraining loop this implies:
periodically (or triggered by an observed precision/recall drop against a refreshed gold-standard
sample) retrain the relevant therapeutic-area adapter on more recent human-adjudicated screening
decisions, version the resulting adapter explicitly (tying directly into Chapter 7's model-version
tracking — an adapter retrain is a new model-version identifier, not a silent in-place update), and
re-validate against the gold-standard set before promoting it into the production screening pipeline.
The same discipline course 11's platform applies to prompt-template and fine-tuning changes generally
— nothing gets promoted on the strength of "it seems better," everything gets checked against a
consistent, held-out evaluation set first.

## Tying It Back

Domain adaptation here means LoRA/QLoRA adapters, one per therapeutic area, trained on that area's
historical human-adjudicated screening decisions — the same mechanism course 11's chapter 02
establishes, applied to screening/extraction reasoning instead of document generation format. Quality
is measured the way real SLR practice measures it: precision and recall against a human-adjudicated
gold-standard subset, with the operating point deliberately biased toward recall because a false
exclusion has no natural downstream catch the way a false inclusion does. And because therapeutic-area
practice and criteria shift over time, adapter retraining is a versioned, evaluated, explicitly
promoted event — never a silent swap — which is exactly the discipline Chapter 7's model-version
tracking depends on to keep "which model produced this screening decision" an answerable question
indefinitely.
