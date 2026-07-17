# 01 — LLM Evaluation Fundamentals

## Why this chapter matters for this project

The resume bullet for this project lists eight evaluation dimensions in one breath: "accuracy,
completeness, hallucination, robustness, efficiency, latency, harmfulness, bad-actor." Before any of
those can be discussed intelligently in an interview, you need a mental model for *why evaluating a
generative model is a fundamentally different problem from evaluating a classifier* — and why a bank's
model risk team can't just ask "what's the accuracy?" the way they might for a fraud-detection model.
This wasn't an academic exercise: the assistant being evaluated here was live in production, answering
real HSBC and Bank of America customers, which is exactly the setting where "what's the accuracy?"
stops being a good enough question.

## Why evaluating generative outputs is hard

Traditional ML evaluation assumes a **single, checkable ground truth**. A fraud classifier predicts
`fraud` or `not fraud`; you compare against a labeled outcome and compute precision/recall. A
generative assistant answering "What's our policy on X?" doesn't have one correct string — it has a
huge space of acceptable phrasings, levels of detail, and framings, several of which might be equally
"correct," and many of which are subtly wrong in ways that are hard to catch mechanically.

This creates three compounding problems:

1. **No single reference answer.** Exact-match or even n-gram overlap metrics (BLEU, ROUGE) — built
   for machine translation and summarization — penalize a perfectly good paraphrase and reward a
   response that happens to reuse the right words while getting the meaning wrong. They are cheap and
   still useful as *one signal among many*, but they cannot carry the whole evaluation.
2. **Failure modes are semantic, not syntactic.** A hallucinated fact, a subtly incomplete answer, or
   a response that technically answers the question but omits a compliance-required caveat all *read*
   fluently. You can't catch them with a spell-checker; you need something that understands meaning.
3. **"Good" is multi-dimensional and audience-dependent.** A response can be accurate but too slow
   (latency), accurate but harmful in tone, or accurate but exploitable (a bad actor could use the
   same phrasing to extract information it shouldn't). A single scalar "quality score" hides which of
   these is failing, which is exactly the information a risk team needs.

This is why the resume bullet doesn't say "measured accuracy" — it lists eight separate metrics. Each
one isolates a different failure mode so it can be tracked, alerted on, and fixed independently.

## The metric taxonomy

Group the eight metrics from the bullet into three families:

**Correctness metrics** — is the content of the response right and sufficient?
- **Accuracy** — are the factual claims in the response actually true (against source docs, a
  knowledge base, or ground truth where one exists)?
- **Completeness** — does the response cover everything the question/task required, or does it stop
  short (e.g., answers part of a multi-part question, omits a required disclaimer)?
- **Hallucination** — does the response assert something that is *not supported* by the provided
  context or retrieved documents? This is related to but distinct from accuracy: a response can be
  factually true in the real world yet still be a hallucination relative to the system's own
  retrieved context, which is the failure mode that matters most for RAG-based assistants (see
  Chapter 02 for the Ragas decomposition of this).

**Operational metrics** — does the system behave well as a piece of infrastructure?
- **Robustness** — does the response quality hold up under paraphrased, noisy, or edge-case input?
  (Chapter 04 covers perturbation testing in depth.)
- **Efficiency** — cost per response: tokens consumed, compute used, retries triggered.
- **Latency** — wall-clock time to respond, often tracked as p50/p95/p99, not just average, because a
  bank cares about the worst case a user experiences, not the typical case.

**Safety metrics** — can the system be misused, or does it cause harm even when used as intended?
- **Harmfulness** — does the response contain toxic, biased, discriminatory, or otherwise
  inappropriate content, or leak PII?
- **Bad-actor resistance** — does the system resist deliberate adversarial manipulation: prompt
  injection, jailbreak attempts, attempts to extract system prompts or training data? (Chapter 04.)

Notice the shape: correctness metrics answer "is it right," operational metrics answer "is it
usable," and safety metrics answer "is it safe to expose." A model risk committee typically wants a
one-line answer in each of these three buckets, not eight separate numbers with no organizing logic —
so structuring your explanation this way (as opposed to reciting eight disconnected terms) reads as
more senior in an interview.

## LLM-as-judge vs. non-LLM metrics

The bullet explicitly says "LLM and non-LLM metrics," which is worth unpacking because it's a real
architectural decision, not just a detail.

**Non-LLM (rule-based / statistical) metrics** are cheap, deterministic, and fast:
- Regex/keyword matching (PII patterns, banned phrases, required disclaimer text present/absent).
- Statistical text metrics (BLEU/ROUGE/BERTScore-style overlap against a reference).
- Classical NLP classifiers (a small fine-tuned toxicity or PII-detection model).
- Straightforward measurements: token count, latency, retrieval hit rate.

Strengths: near-zero marginal cost, fully deterministic (same input always gives same output — useful
for regression testing and for audit trails, since a regulator can be shown exactly why a score was
assigned), and no risk of the *evaluator itself* hallucinating a justification.

Weaknesses: blind to meaning. A keyword filter doesn't know that "the patient should NOT take this
medication" and "the patient should take this medication" differ by one word with opposite clinical
implications.

**LLM-as-judge metrics** use a second LLM call (sometimes the same model, sometimes a stronger or
cheaper specialized one) to *score* the primary assistant's response — e.g., "Given this question,
context, and answer, rate completeness from 1-5 and explain why." This is how most of the semantic
metrics (completeness, hallucination, harmfulness nuance) get evaluated at scale without paying humans
to read every response.

Strengths: captures semantic failure modes non-LLM metrics can't; can produce a natural-language
rationale alongside the score, which is valuable for audit documentation; scales far better than human
review.

Weaknesses: costs real money and adds latency (a second model call per evaluated response, or per
sampled response); the judge model can itself be wrong, biased toward verbose or confident-sounding
answers, or inconsistent across runs unless you fix the prompt, temperature, and rubric carefully;
and — the recursive problem — you now need to validate the judge, which risk teams sometimes push back
on ("who evaluates the evaluator?").

## The practical tradeoff a monitoring system has to make

In production, you rarely run every metric on every response — cost and latency don't allow it. A
realistic design (and the one a Consultant-II would plausibly have proposed here):

- Run **cheap non-LLM checks synchronously on 100% of traffic** — PII/toxicity regex, latency,
  token/efficiency counters, basic guardrail triggers. These gate what reaches the user in real time.
- Run **LLM-as-judge and Ragas-style metrics asynchronously on a sample** (e.g., 5-20% of traffic, or
  100% for a nightly regression suite against a fixed test set) — these feed the dashboard and trend
  lines rather than blocking any single response.
- Treat **robustness and bad-actor testing as offline test suites** run on a schedule or on every
  deploy, not per live response — you're testing the system's behavior under adversarial inputs you
  construct, not scoring organic traffic.

This tiered design is the answer to the interview question "wouldn't running an LLM judge on every
response double your latency and cost?" — you don't; you sample, and you separate real-time gating
checks from asynchronous quality/trend checks. See `notebooks/01_ragas_style_metrics_from_scratch.ipynb`
and `notebooks/02_hallucination_detection_demo.ipynb` for hands-on, from-scratch implementations of the
non-LLM approximations described here, and Chapter 02 for how Ragas formalizes the RAG-specific
subset of these metrics.
