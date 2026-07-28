# 04 — Building the Sequential Pipeline

## Why this chapter matters

Chapters 01–03 covered each stage in isolation. But the résumé bullet's key architectural claim is
"a **sequential architecture**" — the value isn't any one model, it's how OCR, YOLO, and the CNN
compose into a single system, and how you reason about, measure, and debug that system as a whole.
This is the chapter to lean on for any "walk me through the architecture end to end" or "how did you
know it was working" question.

## Architecting a Multi-Stage Sequential Pipeline

The general pattern this project follows — **detect broadly, then narrow, then classify precisely**
— is a common and reusable structure for problems where the target is rare, small, and hard to
define directly from raw input:

```
Stage 1 (OCR)          Stage 2 (YOLO)                Stage 3 (CNN)
high recall,           high recall,                  high precision,
whole-image,           candidate generation           final binary decision
cheap                  over OCR-located regions        per candidate
    |                       |                              |
    v                       v                              v
words + boxes  ------>  candidate boxes  ------>  confirmed superscripts
(everything)            (small, elevated,          (true positives only)
                          near text)
```

Each stage is deliberately given a **narrower, more specific input space than the last**, and each
stage's job gets easier as a result: OCR doesn't need to know anything about superscripts at all —
it just needs to be good at general text detection/recognition. YOLO doesn't need to make a final
true/false call — it just needs to not miss real candidates, working over a much smaller search
space (near-text regions) than the raw image. The CNN doesn't need to search anything — it only
ever sees pre-cropped, pre-localized candidates and makes one binary decision per crop. This
"narrowing funnel" design is why a single end-to-end model is often *not* the first thing to reach
for on a problem like this: it would have to learn text detection, small-object localization, and
fine-grained discrimination all at once, from one loss signal, which is a much harder optimization
problem than training three smaller models each against a task-appropriate objective and dataset.

## Error Propagation Between Stages

The cost of a sequential pipeline is that **errors compound multiplicatively, not additively**, and
every stage after the first can only ever be as good as what the previous stage handed it:

- If OCR **misses** a word or region entirely (a false negative at Stage 1), no downstream stage can
  ever recover it — there's no candidate to even consider. This is why Stage 1 needs to be tuned for
  **recall** (catch everything, even at some cost in precision) rather than precision — false
  positives at this stage are still recoverable downstream; false negatives are not.
- If OCR's bounding box/baseline metadata for a region is **noisy** (Chapter 01's baseline-skew
  failure mode), YOLO's candidate boxes inherit that geometric error, which can shift a true
  superscript's crop just enough to hurt the CNN's classification confidence.
- If YOLO **misses** a true superscript candidate (false negative at Stage 2), the CNN never gets a
  chance to evaluate it, and it's lost — again arguing for tuning Stage 2 toward recall (a lower
  confidence threshold, accepting more false positives) since Stage 3 exists specifically to clean
  those up.
- If YOLO **over-triggers** (many false positives, which Chapter 02/03 established is expected and
  acceptable), the cost shows up only as extra CNN inference calls — a compute/latency cost, not an
  accuracy cost, provided the CNN is precise enough to actually filter them.

The practical implication: **the recall/precision responsibility should be assigned per stage on
purpose.** Stage 1 (OCR) and Stage 2 (YOLO) are tuned to be generous — better to pass along a
borderline candidate than to silently drop a true positive that nothing downstream can ever recover.
Stage 3 (CNN) is where the pipeline earns its precision back. Getting this backwards — e.g., running
YOLO at a high confidence threshold to "clean up" its own output — throws away recoverable true
positives before the stage designed to adjudicate them ever sees them.

## Evaluating the Pipeline as a Whole vs. Each Stage Individually

A common mistake is only ever reporting one end-to-end number. In practice this pipeline needs
metrics at three levels, because a good number at one stage can mask a problem at another:

1. **Per-stage metrics**, evaluated in isolation against stage-appropriate ground truth:
   - OCR: word/character recognition accuracy and detection recall (did it find and correctly
     transcribe the text that's there).
   - YOLO: detection recall and precision at a given IOU threshold (e.g., recall@IOU 0.5) —
     specifically tracking **recall** as the headline number for this stage, per the point above.
   - CNN: classification accuracy, precision, recall, and F1 on the true-superscript-vs-not task,
     evaluated on the actual distribution of candidates YOLO produces (not a clean, balanced
     benchmark set — the real input is heavily imbalanced, as Chapter 02 covered).
2. **End-to-end pipeline metrics**, run on full documents from raw image to final structured
   citation output — this is the number that actually answers "does this system solve the business
   problem," and it's the only number that reflects compounded error propagation across all three
   stages.
3. **Error attribution** — when the end-to-end number is disappointing, tracing *which* stage a
   given failure originated in (missed by OCR? missed by YOLO? misclassified by the CNN?) is what
   makes the difference between being able to say "we improved X" and just re-training everything
   and hoping.

## What Does "Accuracy" Mean Here — 5% to 85%

The résumé's headline number, "citation tracking accuracy," is a claim about the **end-to-end task**,
not any single stage's classification accuracy — and it's worth being precise about that distinction
in an interview, because it's a common follow-up question.

A defensible way to define it: **citation tracking accuracy = the fraction of true citation
markers in a document that were correctly detected, correctly distinguished from look-alike
non-citation text, and correctly attached to their claim text** — i.e., a marker only "counts" as
correctly tracked if it survives all three stages and ends up correctly represented in the final
structured output. This is a stricter, more business-relevant bar than, say, "per-character OCR
accuracy" (which is a Stage 1-only metric and was likely already reasonably high even in the naive
baseline — OCR itself wasn't the broken part) or "per-candidate CNN classification accuracy" (a
Stage 3-only metric that could look good on a balanced eval set while the end-to-end pipeline still
performs poorly if Stage 2's recall is bad).

This framing is also what makes the **5%** baseline plausible rather than suspicious: a naive
"OCR text + regex/heuristic" baseline could easily have reasonable *character*-level OCR accuracy
while still getting the *citation-tracking* task almost entirely wrong, because the heuristic for
"is this a citation marker" — the actual task — was doing nearly all the work, and doing it badly.
The **85%** result reflects the sequential architecture solving that specific gap: high recall
candidate generation (Stages 1–2) combined with precise final adjudication (Stage 3), evaluated on
the full, correctly-defined end-to-end task rather than a proxy metric that would have looked
better than the system actually performed.

## Running This in Production, at Real Document Volume (Eli Lilly / AstraZeneca)

Everything above describes the pipeline as an architecture; in production at Indegene this pipeline
ran daily against real content from two pharma clients, **Eli Lilly** and **AstraZeneca**, deployed
as containerized services on **AWS ECS (Fargate)** behind an **ALB**, with the YOLO and CNN models
hosted as **Sagemaker** endpoints. That deployment reality shapes two decisions worth being explicit
about in an interview.

**Batch vs. real-time on ECS Fargate + Sagemaker.** Citation tracking doesn't need sub-second
latency the way a chatbot or a fraud check does — a document uploaded today needs its citations
tracked before the next MLR review cycle, not before the next HTTP response. That makes **batch
processing the default mode**: documents land in S3, ECS Fargate tasks pull work off a queue and run
Stage 1 (OCR) plus orchestration, and Sagemaker is invoked via **batch transform** jobs (or a small
pool of endpoints scaled to queue depth) rather than an always-on, low-latency endpoint per request.
That's cheaper — you're not paying for idle GPU/CPU capacity between document bursts — and it's a
better fit for the "narrowing funnel" error-propagation model earlier in this chapter, since batch
jobs can afford to retry or re-run a stage without a user waiting on the other end. The exception is
when a client needs a fast turnaround on a small, urgent set of documents (e.g., a last-minute
conference-deck review) — for that case, a lighter-weight **real-time Sagemaker endpoint** path
exists so a handful of pages can be processed in seconds rather than waiting for the next batch
window. Choosing between the two per request is itself an operational decision: default to batch,
escalate to real-time only when a client's SLA demands it, since real-time capacity is the more
expensive of the two to keep provisioned.

**Why 85% (not 5%) was a production requirement, not a nice-to-have.** In a research or demo
setting, a jump from 5% to 85% is just an impressive number. In production, serving Eli Lilly's and
AstraZeneca's actual marketing and scientific content, it's the difference between a system that's
usable and one that isn't: every citation the pipeline tracks (or fails to track) feeds an MLR
compliance workflow, and a pharma client cannot ship promotional material with an unsubstantiated or
mis-linked claim. At 5% accuracy, the system would generate so many missed and mislabeled citations
that a human reviewer would need to re-check essentially everything by hand — at which point the
pipeline adds latency and cost without meaningfully reducing review burden, which is not something
you can put in front of a paying enterprise client. At 85%, the system is accurate enough that human
review time shifts from "check everything" to "check the flagged/low-confidence subset," which is
what makes automated citation tracking a viable production service rather than a research result.
That's also why the per-stage and end-to-end observability discussed above (recall/precision per
stage, drift signals, downstream MLR override rates) isn't optional in this deployment — with real
compliance consequences riding on the output, silently regressing from 85% back toward the 5%
baseline has to be caught by CloudWatch-backed monitoring before a client ever notices it in their
review queue, not after.

## Tying It Back

"Sequential architecture" in the résumé bullet is the load-bearing phrase for this whole course:
three models, each with a deliberately different recall/precision responsibility, chained so each
stage's output becomes the next stage's input space, evaluated both per-stage (to know where to
improve) and end-to-end (to know if the business problem — accurate automated citation tracking —
is actually being solved). That's the story to tell when asked to describe the architecture from
memory, and it's the reasoning behind why 5% -> 85% is a citation-tracking-task number, not a
model-accuracy number.
