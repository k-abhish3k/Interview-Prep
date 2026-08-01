# 07 — Model and Criteria Staleness

## Why this chapter exists

This course's version of the curriculum's now-familiar "how do you handle a revised version of
something" chapter has a twist the other courses don't: there are genuinely **two independent things
that go stale here**, not one. Course 11's chapter 06 tracks a single axis (the clinical Protocol's
version). Course 8 tracks vector-index staleness against source-document changes. This project has to
track **both** which model version screened a given abstract **and** which criteria version it was
screened against — and those two can drift completely independently of each other. Getting that
distinction right, and building a design that tracks both explicitly rather than assuming one implies
the other, is the substance of this chapter.

**The same honesty note this curriculum applies everywhere it reconstructs a design**: there's no
source repository behind this course. Everything below — the naive assumption, the gap, the proposed
reconciliation design — is a plausible, technically detailed reconstruction, written with the same
confidence and specificity as course 11's chapter 06 so it reads as a strong interview answer, not a
verified account of a real implementation.

## Part 1 — The naive assumption

A systematic literature review runs over weeks or months. During that window, two kinds of change can
happen to the process screening abstracts:

1. **The model changes** — a new DeepSeek release lands, or a therapeutic-area LoRA adapter gets
   retrained on fresher gold-standard data (Chapter 5's retraining loop).
2. **The criteria change** — a review's inclusion/exclusion criteria get refined mid-review. This is
   not a rare edge case in real SLR methodology: it's common for criteria to be clarified or narrowed
   after initial screening has already begun, as the review team encounters ambiguous cases the
   original criteria wording didn't anticipate, or as a protocol amendment adjusts scope.

The naive assumption a first-pass implementation tends to make, implicitly rather than by deliberate
design: **screening decisions made under model/criteria version N remain valid indefinitely** — once
an abstract has been screened and its include/exclude verdict recorded, nothing about that record
ever needs revisiting, regardless of what changes about the model or the criteria afterward. That
assumption is comfortable precisely because it requires no additional bookkeeping — the moment you
start asking "but is this verdict still valid under the *current* criteria," you need infrastructure
that a naive first pass usually doesn't build.

## Part 2 — The gap

The gap is exactly where that comfortable assumption breaks: **a criteria refinement without
re-screening the abstracts already processed under the old criteria creates an inconsistent review**
— some abstracts screened under the original criteria wording, some under the refined wording, with
no clear record distinguishing which is which. Concretely: a review begins with an inclusion
criterion of "randomized controlled trials, any phase," screens three thousand abstracts against
that wording, and then the review team clarifies the criterion to "randomized controlled trials,
Phase II or later" after noticing several Phase I trials were being inconsistently included. If the
system has no record of *which criteria version* screened each of those three thousand abstracts,
there's no way to identify which of them need to be re-run against the refined wording — some Phase I
trials from before the refinement might still sit in the "included" set, silently inconsistent with
every abstract screened after the refinement landed.

That's a genuinely severe failure mode for a process whose entire premise is methodological rigor: a
PRISMA-style SLR report is expected to describe one coherent screening process applied consistently
across the corpus, not an undocumented mixture of two different standards applied to different
subsets of it, with nobody able to say afterward which subset got which treatment.

## Part 3 — Two concerns that must not be conflated

This is the crux, and it's the reason this chapter opened by calling out that this project tracks two
axes rather than one:

- **Concern A: "the model version used is current."** This is about whether the *inference engine*
  producing screening decisions reflects the intended, validated model artifact — a specific,
  provenance-verified (Chapter 6), gold-standard-evaluated (Chapter 5) DeepSeek checkpoint plus
  therapeutic-area adapter. A screening decision made under a superseded model version isn't
  necessarily *wrong* — the prior model may have been perfectly adequate — but it was made under a
  different, now-superseded standard of screening quality, which matters if that model version is
  later found to have a systematic issue (e.g., a precision/recall regression Chapter 5's monitoring
  catches after the fact).
- **Concern B: "the criteria version a given screening decision was made against is current."** This
  is entirely separate — it's about whether the *inclusion/exclusion standard itself* being applied
  matches the review's current, intended definition of what belongs in scope. A screening decision
  can be made by the exact right, current model version and still be **stale relative to the
  criteria**, if the criteria have moved on since that abstract was screened.

These can drift **independently**, and that independence is the whole point: a review could be
running the correct, current model version throughout (Concern A fully satisfied) while still
carrying thousands of screening decisions made against superseded criteria wording (Concern B
silently broken) — because a model-version update and a criteria refinement are two completely
different events, triggered by different people, on different timelines, with no inherent
relationship to each other. Exactly the same category of mistake course 11's chapter 06 warns against
for its own two concerns (SOC accuracy vs. dependent-document staleness) — a system, or an
interviewer's question, that only asks about one of these two axes can get a clean answer while the
other axis is silently broken, and a strong answer here names both explicitly rather than letting one
stand in for both.

## Part 4 — A proposed design: dual-tagging and reconciliation

**Every screening decision records both a `model_version` and a `criteria_version` identifier at the
moment it's made** — not derived after the fact, not inferred from a timestamp, but stamped directly
onto the record as it's written, the same way course 06's `protocol_version` field is stamped onto
every generated document at creation time.

```python
# Proposed data model — illustrative, not implemented

@dataclass
class ScreeningDecision:
    abstract_id: str
    review_id: str
    model_version: str        # e.g. "deepseek-v3-oncology-lora-2026.03"
    criteria_version: int     # monotonically increasing per review, like protocol_version
    verdict: str              # "include" | "exclude"
    exclusion_reason: str | None
    screened_at: datetime
    review_status: str        # "auto_screened" | "human_confirmed" | "needs_rescreen"
```

**A reconciliation pass triggers whenever a review's criteria change**, mirroring course 11's
`flag_stale_dependents` mechanism but applied to the criteria axis instead of the protocol-version
axis:

```python
def flag_stale_screening_decisions(
    review_id: str,
    current_criteria_version: int,
    decisions: list[ScreeningDecision],
):
    """Run whenever a review's inclusion/exclusion criteria are refined. Any screening
    decision whose criteria_version trails the review's current criteria_version is not
    wrong -- it's stale relative to the new standard, and staleness is a distinct,
    first-class status that must be visible, never silently indistinguishable from
    'screened under the current criteria'."""
    flagged = []
    for d in decisions:
        if d.review_id == review_id and d.criteria_version < current_criteria_version:
            d.review_status = "needs_rescreen"
            flagged.append(d.abstract_id)
    return flagged
```

A parallel, independent check runs for `model_version` drift — flagging screening decisions made
under a superseded model version for at least a spot-check re-review, even when the criteria haven't
changed, since Concern A and Concern B are tracked, flagged, and resolved through separate mechanisms
that happen to share the same underlying record structure. A flagged `needs_rescreen` abstract is
either automatically re-screened against the current criteria version and current model version (the
common case, since re-screening a single abstract is cheap relative to a full corpus re-screen) or, if
the reconciliation pass surfaces a large enough flagged set to warrant it, routed to a human reviewer
for direct re-adjudication rather than blindly trusting an automated re-run.

Critically — matching course 11's chapter 06, Part 5's insistence that a version bump is never a fast
path around review — a re-screened abstract's new verdict goes through exactly the same downstream
human-confirmation gate (Chapter 6, Chapter 8) a first-time screening decision would, never an
auto-accepted status just because "it's just a criteria update." A criteria refinement is very often
exactly the moment a screening decision most needs a fresh, careful look, not the moment to fast-track
it through.

## Part 5 — The full interview-ready answer, assembled

> "This platform tracks two things that can go stale independently, and it's important not to
> conflate them: whether the model version used to screen an abstract is the current, validated
> version, and whether the criteria version that abstract was screened against is the review's
> current, intended standard. A model update and a criteria refinement are two completely different
> events on two completely different timelines, so a review can have a perfectly current model
> while carrying thousands of screening decisions made against superseded criteria, or vice versa.
> The design I'd use tags every screening decision with both a model_version and a criteria_version
> identifier at the moment it's made, and runs a reconciliation pass whenever either changes — any
> decision trailing the current version on either axis gets flagged needs_rescreen, either
> auto-re-screened or routed to a human reviewer, and the re-screened result goes through the exact
> same human-confirmation gate a first-time screening decision would. The goal isn't to promise every
> decision is always current — it's to make staleness on either axis a visible, tracked,
> reconcilable state instead of an invisible inconsistency nobody can detect after the fact."

## Tying It Back

The strongest thing in this chapter is the same kind of clean separation course 11's chapter 06 built
for its own two concerns, applied to a genuinely different pair of axes: model-version currency and
criteria-version currency drift independently, get flagged independently, and get reconciled through
the same dual-tagging-plus-reconciliation-pass mechanism, without letting either one hide behind the
other. That design leans directly on Chapter 6's provenance work — a `model_version` identifier is
only a meaningful tag if it traces back to a verified, checksummed artifact — and it's the concrete
mechanism that makes Chapter 2's regulatory-auditability argument operational: a reviewer, or a
regulator, asking "what standard was this specific screening decision made under" gets a precise,
two-part answer, not a shrug.
