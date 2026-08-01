# 06 — Development Challenge: LLM Evaluation Before Production

## The problem: proving "safe enough to ship" with no ground truth

Once a GenAI feature works in a demo, the actual hard problem starts: convincing yourself — and a
bank's model-risk function or a pharma client's medical/legal/regulatory (MLR) reviewers — that it's
safe and accurate enough to put in front of real customers. Unlike a classification model with a
labeled test set and an unambiguous accuracy number, an open-ended generative feature has no single
ground truth to check against, an effectively unbounded input space, and failure modes (a confident
but wrong answer, a subtly off-policy claim, a tone that's technically correct but inappropriate)
that a simple pass/fail test doesn't capture well. This is the chapter for "how did you know it was
ready to ship" — a favorite follow-up after any GenAI project story, and a direct predecessor to
chapter 03's production monitoring (the golden eval set built here is the same asset the pre-deploy
gate in chapter 03 runs against on every subsequent change).

## Building a golden evaluation set with no pre-existing ground truth

The standard fix for "no ground truth" is to build one deliberately, sized and shaped for the
specific risk profile of the feature:

- **Curate real (or realistic, de-identified) examples across the input space**, weighted toward
  the cases that matter most: common queries, edge cases, known-hard categories (multi-part
  questions, ambiguous phrasing, adversarial/prompt-injection-style inputs), and — critically —
  cases the client's own subject-matter experts flag as high-stakes if answered wrong (a claim about
  drug efficacy, a statement of bank policy that carries regulatory weight).
- **Get subject-matter-expert-authored or expert-reviewed reference answers or acceptance criteria**
  for each case — not necessarily a single "correct" string (generative outputs are rarely
  string-matchable), but a rubric: what must the answer contain, what must it not claim, what
  source must it be grounded in.
- **Score with a mix of cheap automated checks and LLM-as-judge evaluation** (course 4's
  hallucination/robustness/harmfulness metrics are exactly this pattern), reserving the most
  expensive step — human expert review — for the highest-stakes subset and for periodically
  auditing the LLM-judge's own agreement with human judgment, since an LLM judge can itself be
  miscalibrated.
- **Keep the golden set versioned and growing** — every production incident (chapter 03) that
  reveals a new failure mode should add a regression case to this set, so the same mistake can't
  silently ship again. (See `notebooks/03_llm_regression_testing_harness.ipynb` for a runnable,
  fully-offline example of exactly this pattern: a golden set plus a harness that flags regressions
  between two prompt/model versions.)

## Catching hallucination and harmful-output risk before launch, not after

Pre-launch evaluation for a bank or pharma client needs to specifically probe for the failure modes
those clients care most about, not just general answer quality:

- **Groundedness/faithfulness testing**: does every factual claim in the answer trace back to a
  real source document, or does the model fill gaps with plausible-sounding invention? This is
  tested by deliberately including golden-set cases where the correct answer is "I don't know" or
  "this isn't covered in the available documents" — a model that always tries to answer, even when
  it shouldn't, is a hallucination risk hiding behind apparent helpfulness.
- **Adversarial/robustness testing**: rephrased, ambiguous, or deliberately misleading versions of
  the same underlying question, to check the model doesn't give materially different (and
  potentially contradictory) answers to the same real intent — a robustness failure mode that's
  easy to miss if the golden set only has one phrasing per question.
- **Harmful/off-policy output testing**: prompts designed to probe whether the model can be steered
  into giving financial advice it isn't licensed to give, medical claims that require MLR
  substantiation it hasn't gone through, or content outside its intended scope — the "bad-actor"
  category from course 4's metric set applied at pre-launch time rather than only in production.

## Getting compliance/legal sign-off

For a banking or pharma client, a golden eval set and a good demo aren't sufficient to launch —
there's a governance step where a model-risk or MLR reviewer has to sign off, and that step goes
much faster if the evaluation artifact is built with their concerns in mind from the start rather
than retrofitted: a written evaluation methodology, the golden set itself (available for their own
spot-checking), pass/fail thresholds agreed in advance rather than negotiated after seeing results,
and a clear owner for ongoing monitoring once live (handing off directly into chapter 03's
production observability). This is the same governance muscle described in course 4's SR 11-7
context, applied at the pre-launch gate rather than only as an ongoing production discipline.

## Illustrative struggle: it looked good in demos, then failed the harder eval set

> **Illustrative, plausible scenario — not a verified account of a specific project's development
> history.** Built from the golden-eval-set methodology described above.

**Situation.** A GenAI feature (in the shape of course 8's virtual liaison platform, built on
RAG with LangChain/Pinecone for a pharma client) performed well in internal demos and initial
stakeholder walkthroughs — the demo queries were mostly clean, well-formed questions close to the
documents in the retrieval index, and the answers looked accurate and well-cited.

**Task.** Before this could go in front of real end users, I needed a rigorous pre-launch
evaluation that went beyond demo-quality queries, specifically because the client's MLR reviewers
required evidence the system had been tested against the harder, messier queries real users would
actually type — not just the clean queries the demo had been curated around.

**Action.** I built a golden evaluation set deliberately including categories the demo hadn't
covered: multi-part questions spanning more than one retrieved document, ambiguously phrased
questions, questions with no good answer in the index (to test whether the system would correctly
decline rather than hallucinate), and adversarial rephrasing of the same question to check answer
consistency. Running the existing system against this harder set surfaced a real gap: on
multi-part questions, the retrieval step was only pulling context for the first part of the
question, and the model was answering confidently on the second part using general knowledge
rather than the client's actual documents — a groundedness failure invisible in the single-part
demo queries. I fixed the retrieval step to decompose multi-part questions before retrieval so each
sub-question got its own grounded context, re-ran the full golden set to confirm the fix didn't
regress the cases that had previously passed, and shared the before/after golden-set results with
the MLR reviewers as the evidence package for sign-off.

**Result.** *(Illustrative)* The groundedness failure rate on multi-part questions dropped from a
meaningful fraction of the golden-set cases (illustrative: roughly 25%) to near zero after the
retrieval fix, and the MLR review proceeded on the strength of the golden-set evidence rather than
requiring a second, slower manual audit round. The multi-part-question category was retained
permanently in the golden set as a standing regression check for future prompt/retrieval changes.

## Talking points this chapter sets up

- Building ground truth deliberately (curated golden set + expert rubric) when none exists, rather
  than treating "no ground truth" as a reason to skip rigorous evaluation.
- The specific failure categories that matter for regulated clients: groundedness, robustness to
  rephrasing, and harmful/off-policy output — not just generic answer quality.
- Why demo-quality queries systematically under-test a system, and what a harder, deliberately
  messier golden set catches that a demo can't.
- The golden set as a durable artifact that both unblocks compliance sign-off pre-launch and powers
  the regression gate referenced in chapter 03 for every change after launch.
