# 99 — Interview Q&A: Model Risk Monitoring for AI Assistant

Read this last, after Chapters 00-07, while the concepts are fresh. Questions are grouped
behavioral -> technical deep-dive -> system design -> reflective. Practice the STAR question out loud
in under 90 seconds before anything else. **Question 2 is the "gotcha" question — the natural, sharp
follow-up any interviewer who understood chapter 05's rolling-baseline alerting would ask next** — it's
positioned right after the behavioral warm-up, deliberately, the same way Course 03 puts its real
asked-in-an-interview question first. Questions 3-5 are direct follow-ups drawing on chapter 07.

---

### 1. (Behavioral, STAR) "Tell me about a project where you had to evaluate an AI system's quality or safety."

**Model answer:**
"At Capco, I worked on a Model Risk Monitoring layer for a generative AI Assistant we'd built for a
banking client. Because the client is a regulated financial institution, their model risk function
required ongoing, documented evidence the assistant was accurate and safe before it could stay in
production — that's standard under SR 11-7-style model risk frameworks in banking. My task was to
design and build the evaluation logic: metrics for accuracy, completeness, hallucination, robustness,
efficiency, latency, harmfulness, and resistance to adversarial ('bad-actor') input, combined with
real user feedback and engagement signals. I built an evaluation harness combining cheap non-LLM
checks that ran on every response with LLM-as-judge and Ragas-style metrics run on a sample, added
explainability with SHAP/LIME/Shapash so flagged decisions were auditable, and fed everything into a
Power BI/Tableau dashboard with threshold-based alerting. The result — *illustrative, replace with your
real number* — was catching an estimated 85% of hallucinated responses before they reached end users
unflagged, and moving detection from a manual weekly review to near real-time alerting."

*Follow-up to expect: "What was the hardest part?"* — Have a specific, honest answer ready, e.g.
tuning alert thresholds so they weren't too noisy on low-traffic periods, or getting an LLM-as-judge
metric to be consistent run-to-run.

---

### 2. If the chatbot's prompt or model changes, how does your monitoring system avoid comparing against a stale baseline?

This is the gotcha question — the analog, for this project, of the real "how do you handle revised
versions of the same document" question Course 03's uploader service was actually asked. The full worked
answer lives in chapter 06 (`06-baseline-staleness-and-drift-lifecycle.md`) — this is the compressed,
interview-ready version:

> "Today, the rolling baseline — a 30-day trailing mean and standard deviation, per chapter 05's alerting
> design — has no concept of a version boundary. There's a deployment marker drawn on the executive trend
> chart, but it's a chart annotation for a human to eyeball, not an input the alerting engine itself
> reads. So a deliberate prompt or model change either looks like a false-positive regression for as long
> as pre-change data still dominates the trailing window, or — the worse direction — a genuine regression
> slowly gets absorbed into the baseline as 'the new normal' over that same trailing window, quietly
> loosening the threshold around exactly the failure it should be catching.
>
> The workaround that exists today is entirely manual: silence alerting for a known deploy window, then
> manually reset the baseline's lookback so it starts from the deploy date instead of waiting up to 30
> days for old data to roll out naturally. That only works if whoever's shipping the change remembers to
> do both steps — nothing technical enforces it, and a hotfix or a silent model-provider-side update can
> ship with neither step happening at all.
>
> It's also worth being precise that this is a different problem from genuine drift. A stale knowledge
> base, a shifting user population, or even a silent update on the model provider's side are real
> degradations the baseline *should* catch — that's the system working as intended. Conflating that with
> 'someone must have deployed something' and resetting the baseline in response would quietly launder a
> real regression into the new normal, which is the worse of the two mistakes.
>
> If I were building a real fix, I'd version the baseline itself: tag every response with a fingerprint
> of the active prompt/model/retrieval config — an `assistant_version_tag` — and key each client's
> baseline to `(client_id, assistant_version_tag)` instead of just `client_id`. A new tag opens a
> `LEARNING`-phase baseline that still lets alerts fire (at lower urgency, not silenced) against the
> previous version's baseline until the new one accumulates enough samples to stabilize, then flips to
> `ACTIVE` and retires — not deletes — the old one, so a clean before/after comparison stays possible.
> But that version tag can't be reliably inferred from the metrics side, because a distribution shift is
> exactly what it's supposed to help classify — it has to come from the CI/CD pipeline that actually
> deploys the assistant, as a release-gate event, the same way Course 04's pipeline would be the natural
> place to emit it."

**Follow-ups to be ready for (drawing on chapter 07):**

---

### 3. Two of your async evaluators flag the same conversation independently — how do you know you're not double-counting it?

**Model answer:** By construction, this is a real race: a synchronous Tier 1 rule-based gate and an
asynchronous Tier 3 LLM-as-judge classifier can both independently decide to flag the same
`response_id`, running on separate execution paths precisely because they're not supposed to block each
other. A naive application-level "read the current count, write count + 1" pattern is not safe under
that concurrency — it can lose an increment, or in some interleavings double-count one flagged
conversation as two. The fix is to stop incrementing a counter in application code at all: use an atomic
SQL increment, or better, maintain a `flags` table with a unique constraint on `(response_id,
evaluator_name)` and count distinct flagged `response_id`s. That makes a single evaluator's retried write
idempotent, and two *different* evaluators flagging the same response correctly collapses to one flagged
conversation, not two. This is exactly the kind of bug that would only show up under genuine concurrent
load, not a sequential unit test — a real regression test for it has to fire both evaluators against the
same response truly concurrently and assert the final count.

---

### 4. Which evaluator failures are you comfortable letting block a live response, and which should never block?

**Model answer:** The split is entirely about which tier the evaluator runs in, not how "important" the
metric sounds. Only the synchronous Tier 1 gate (cheap PII/toxicity/injection-pattern checks, on 100% of
traffic) is in a position to block anything, because it's the only evaluator still in the request path
when the decision to release a response gets made — and even there, it fails closed on an actual
exception (a bug shouldn't silently let something unsafe through) but fails open on merely exceeding its
latency budget (a rare slow evaluation shouldn't take down the whole assistant's availability). Anything
past Tier 1 — LLM-as-judge, Ragas metrics, the hourly aggregation job — runs after the response has
already reached the user, so by definition it can never block that response; a failure there just gets
logged and excluded from that period's aggregate, distinctly from a genuine low score, so a spike in
evaluation failures is itself visible rather than silently treated as "everything's fine."

---

### 5. What's the one hardening gap you'd flag first if a security reviewer asked about this platform?

**Model answer:** The alert-routing webhook URL — the endpoint that posts critical-severity pages into a
bank's on-call channel — being a bare, unauthenticated URL sitting in plain configuration rather than a
secret store. Incoming webhook URLs of that style are effectively bearer tokens: anyone who can read that
configuration value can post directly into the channel with no further authentication, which means a
leaked URL could be used to flood a bank's on-call rotation with fabricated pages (training the team to
distrust real ones) or simply to learn where a client's alerts are routed. The fix is the same shape as
the hardcoded session-secret gap in Course 03's uploader service: move it into Key Vault, reference it at
runtime, and rotate it per client so a compromise on one bank's channel doesn't also expose the other's.

---

### 6. How do you measure hallucination without ground-truth labels?

**Model answer:** Ground truth for "is this true" is expensive and often doesn't exist for novel user
questions, so instead of checking truth against the real world, you check the response against the
system's own retrieved context — that's Ragas's **faithfulness** metric. You decompose the answer into
individual claims and check whether each is entailed by the retrieved context, typically using an LLM
judge (or a lighter NLI/entailment model for cost) rather than requiring a labeled reference answer.
This reframes hallucination as "unsupported by evidence the system itself had access to" rather than
"factually false in an absolute sense" — which is both more tractable to measure and, for an
audit trail, more defensible, because you can point to the specific unsupported claim and the context
that failed to support it.

---

### 7. What's the difference between LLM-as-judge and Ragas metrics — aren't they the same thing?

**Model answer:** They overlap but aren't identical. LLM-as-judge is a *general technique* — use an
LLM to score another LLM's output against some rubric (accuracy, completeness, tone, whatever you
define). Ragas is a *specific framework* that applies LLM-as-judge (and some embedding-similarity
techniques) to a fixed, well-defined set of RAG-specific metrics — faithfulness, answer relevancy,
context precision, context recall — each isolating one link in the retrieve-then-generate chain. So
Ragas metrics are usually implemented *using* LLM-as-judge under the hood for some of them
(faithfulness's claim-support check), but LLM-as-judge is the broader technique and Ragas is one
opinionated, RAG-specific application of it. You could build a custom LLM-as-judge rubric for
something Ragas doesn't cover — say, "tone appropriateness for a banking context" — that wouldn't be
a Ragas metric at all.

---

### 8. How would you detect a prompt injection attack in production?

**Model answer:** Layered, not single-point. On the input side: pattern/classifier-based detection of
common injection phrasing ("ignore previous instructions," role-play jailbreak framing) to flag or
route suspicious queries differently. Structurally: treat any retrieved or user-supplied content as
untrusted data, not instructions — wrap it in clear delimiters and explicitly instruct the model not
to follow instructions found inside that data. On the output side: a second pass (classifier or
lightweight LLM judge) checking whether the response complied with the *original* system instructions
or appears to have been hijacked by embedded content — e.g., did it start doing something unrelated to
the user's actual question right after processing a retrieved document. In monitoring terms, this
becomes a **bad-actor flag rate** metric tracked on the dashboard, plus an offline regression suite of
known injection patterns run on every deploy so a previously-caught attack can't silently start
working again after a prompt or model change.

---

### 9. SHAP vs. LIME — when would you pick one over the other?

**Model answer:** Both explain individual predictions, but differently. SHAP is grounded in Shapley
values from game theory, gives theoretically consistent local *and* aggregatable global explanations,
and is fast/exact for tree-based models via TreeSHAP — I'd reach for it when I need defensible,
auditable explanations for something like a harmfulness or bad-actor classifier, especially if it's
tree-based, and when I also want a global feature-importance view for model documentation. LIME is
model-agnostic, cheaper to run, and purely local — it fits a simple surrogate model around one
prediction by perturbing the input and observing the output change. I'd reach for LIME for quick,
model-agnostic debugging of one specific flagged case, or when the underlying model doesn't have a
good SHAP explainer available. The tradeoff is LIME explanations can be less stable run-to-run because
of the random sampling in the perturbation step, whereas SHAP is more consistent but costs more compute.

---

### 10. Where does Shapash fit if SHAP and LIME already exist?

**Model answer:** Shapash isn't a competing algorithm — it's a reporting/dashboard layer built on top
of SHAP (and LIME) output, designed for non-technical stakeholders. In a model-risk context, that
audience — risk officers, compliance reviewers — is exactly who needs to consume explanations, and
they're not going to read raw SHAP value arrays. Shapash turns that into human-readable feature
labels, contribution plots, and an interactive web dashboard reviewers can browse without writing
code. It's the piece that turns "we have explainability" into "we have a reviewable artifact we can
hand to an auditor."

---

### 11. Why track both efficiency and latency separately — aren't they the same thing?

**Model answer:** They're correlated but measure different failure modes. Latency is wall-clock time
to respond — what the user experiences, usually tracked as p50/p95/p99 rather than average, because a
bank cares about the worst-case tail, not the typical case. Efficiency is closer to cost — tokens
consumed, retries triggered, compute used per response. A response can be fast but expensive (e.g., it
succeeded on the first try but used a very long context window), or slow but cheap (throttled by rate
limits rather than heavy compute). Tracking them separately lets you diagnose *why* something is
degrading — a latency spike with flat efficiency points to an infrastructure/network issue; an
efficiency spike with flat latency points to prompt bloat or excessive retries.

---

### 12. How do you decide what to run synchronously (blocking) vs. asynchronously (sampled) in the evaluation pipeline?

**Model answer:** Split by cost and by what the check is protecting against. Anything cheap and
deterministic that gates *safety* — PII regex, toxicity keyword filters, basic guardrail triggers —
runs synchronously on 100% of traffic because it needs to block a bad response before it reaches the
user. Anything expensive (LLM-as-judge, Ragas metrics) that mainly feeds *quality trend tracking*
rather than blocking a specific response runs asynchronously on a sample, or offline against a fixed
regression test set on every deploy. Robustness and bad-actor testing are almost always offline test
suites, not per-response checks, because you're testing behavior against constructed adversarial
inputs, not scoring organic traffic. The overall principle: real-time gating for safety-critical,
cheap checks; sampled/async for expensive, quality-trend checks.

---

### 13. What's the difference between robustness testing and adversarial ("bad-actor") testing?

**Model answer:** Different threat models. Robustness testing assumes a *good-faith* user who happens
to phrase things differently — typos, paraphrases, casual register — and checks whether answer quality
holds steady across those natural variations; a failure here means the system is brittle, not
malicious. Adversarial/bad-actor testing assumes a user (or embedded content) *deliberately* trying to
manipulate the system — prompt injection, jailbreak framing — and checks whether the system resists
that manipulation. Both are implemented similarly (a curated test set run against the assistant, with
pass/fail or score-delta grading), but robustness failures usually get fixed with better retrieval or
prompt clarity, while bad-actor failures get fixed with guardrails, structural input/output
separation, and stricter refusal behavior.

---

### 14. How do "reliance" and "engagement" fit into a metrics pipeline that's mostly about correctness and safety?

**Model answer:** They're the implicit-signal counterpart to the explicit metrics. Faithfulness,
harmfulness, etc. are computed *by the system evaluating itself* — reliance (did the user act on the
answer, retry, or immediately rephrase) and engagement (session length, return usage) are signals from
*real user behavior* that can catch problems the automated metrics miss. For example, a response can
score well on every automated metric and still get immediately rephrased by users on a specific
topic — that's evidence of a quality problem the metrics didn't capture, maybe a tone or usefulness
issue rather than a faithfulness issue. None of these signals is reliable alone (feedback is sparse,
reliance is an indirect proxy, engagement is aggregate-only), so they're tracked alongside, not instead
of, the explicit metrics, and a persistent mismatch between "metrics look fine" and "reliance/
engagement looks bad" is itself a useful finding.

---

### 15. (System design) Design a real-time moderation pipeline that doesn't add unacceptable latency.

**Model answer sketch:**
- **Tier 1 — inline, blocking, sub-50ms budget:** cheap deterministic checks only — regex PII/toxic-
  keyword filters, a small distilled classifier for obvious harmful content, input-side injection
  pattern matching. Runs on 100% of traffic, in the request path, before the response is returned.
- **Tier 2 — inline, blocking, but on the *output* only if Tier 1's input check flagged something
  suspicious:** for the (hopefully small) fraction of flagged traffic, pay the cost of a heavier check
  (a fast LLM-as-judge call, or a stricter classifier) before releasing the response, accepting the
  added latency only for the flagged minority rather than every request.
- **Tier 3 — asynchronous, non-blocking:** the full metric suite (Ragas metrics, deeper LLM-as-judge
  scoring, SHAP-based explanation generation) runs after the response has already been returned to the
  user, on a sample of traffic, writing to the metrics store for dashboards/alerts. This is where most
  of the "expensive" evaluation happens without touching user-facing latency at all.
- **Alerting:** threshold breaches from Tier 3 aggregation trigger dashboard alerts, not per-response
  blocking — by the time Tier 3 catches something, the individual response has already gone out, so
  the value is in *trend* detection (catching a regression fast) and feeding back into Tier 1/2 rule
  updates, not blocking that one response after the fact.
- **Key tradeoff to name explicitly:** you're accepting that a small number of bad responses reach
  users before the full evaluation catches the pattern — that's the honest cost of not running
  everything inline, and it's why Tier 1's cheap filters need to be tuned conservatively (favor
  recall over precision) even though they'll have more false positives, because Tier 1 is the only
  layer that can stop a bad response before it's seen.

*Good follow-up to anticipate: "What if Tier 1's classifier itself is slow?"* — mention distillation
or caching common patterns, and that Tier 1 budget (e.g., 50ms) is itself a tunable SLO decided with
the product/latency requirements, not a fixed law.

---

### 16. (System design) How would you validate that your LLM-as-judge metric is itself trustworthy?

**Model answer:** Treat the judge like any other model that needs validation, not as ground truth by
assumption. Practical steps: (1) build a small human-labeled gold set (even 100-200 examples) and
measure agreement between the LLM judge's scores and human ratings — report something like Cohen's
kappa or simple accuracy against the human labels; (2) check for known judge biases — length bias
(judges often rate longer answers as more complete regardless of quality), position bias in pairwise
comparisons, and self-preference bias if the judge model is the same family as the model being judged;
(3) fix the judge's prompt, temperature (near 0), and rubric, and version them — an unversioned judge
prompt makes metric trends over time meaningless because you can't tell if a score change reflects the
assistant or the judge changing; (4) periodically re-run the human-agreement check as a regression test
on the judge itself, especially after switching judge model versions.

---

### 17. What would you change if you rebuilt this today?

**Model answer (adapt to your real experience, but structure to use):** "A few things. First, I'd
invest earlier in the human-labeled gold set for validating the LLM-as-judge scores — we built the
judge-based metrics before we had a rigorous way to confirm the judge itself was reliable, which meant
early metric trends were harder to trust with full confidence. Second, I'd design alert thresholds as
rolling/adaptive from day one instead of static — static thresholds either stayed silent on real
regressions during low-traffic periods or were noisy during high-traffic ones, and adaptive baselining
would have caught that earlier. Third, I'd push harder for the bad-actor/adversarial test suite to be
a required gate in the deployment pipeline rather than a periodic check, so a previously-fixed
injection vulnerability couldn't silently regress after a prompt change. None of these are things that
were done wrong exactly — they're the kind of maturity a system like this accrues over its first few
months in production, and I'd try to front-load them next time."

---

### 18. Why does a bank need a formal model-risk process for a chatbot — isn't that overkill compared to a normal software bug?

**Model answer:** A traditional software bug is usually deterministic and reproducible — the same
input produces the same wrong output, and you can write a regression test that permanently fixes it.
A generative model's failure modes are probabilistic, can regress silently after a prompt, retrieval,
or base-model change with no code change at all, and its outputs directly influence what an employee
or customer believes or does — closer to a decision-support system than a typical CRUD bug. Regulators
treat models this way deliberately: under SR 11-7-style frameworks, a model that materially influences
decisions carries risk that scales with usage, not with lines-of-code changed. So the formal process —
independent validation before launch, ongoing monitoring, documented explanations for flagged
decisions, a named owner accountable for the model's behavior — isn't bureaucratic overkill relative
to a chatbot's apparent simplicity; it's proportional to the fact that a chatbot answering "what's our
refund policy" wrong at scale, silently, for weeks, is a real operational and reputational risk to a
bank in a way a typical UI bug isn't.

---

### 19. How do you handle disagreement between the automated metrics and user feedback — e.g., faithfulness scores are high but thumbs-down rate is rising on a topic?

**Model answer:** Treat it as a signal that the automated metrics are measuring the wrong thing for
that segment, not that user feedback is noise to be dismissed. First step is manual case review on the
drill-down dashboard — read a sample of the thumbs-down responses directly rather than trusting either
signal blindly. Common root causes in this pattern: the response is faithful to the retrieved context
but the context itself is stale or the wrong document (a context-precision problem the faithfulness
metric wouldn't catch, since faithfulness only checks answer-against-context, not context-against-
reality); a tone or format issue that isn't a "correctness" problem at all (users perceive the
response as unhelpful or condescending even though every claim is technically supported); or a gap
between what the automated rubric asks the judge to check and what users actually care about, which
usually means it's time to revisit and expand the judge's rubric or add a new explicit metric rather
than relying on faithfulness/relevancy alone to represent overall quality.

---

### 20. You built this for two banking clients, HSBC and Bank of America — how did you guarantee metric/data isolation between them?

**Model answer:** Two layers, deliberately redundant. At the data layer, every record written into the
metrics store — every per-response evaluation, every feedback signal — carries a mandatory `client_id`
set at ingestion by the evaluation harness itself, never inferred or defaulted downstream; a record
missing it fails loudly rather than being silently attributed to the wrong client. At the resource
layer, rather than relying only on a filter inside a shared dashboard, each client's telemetry lived in
its own Azure Monitor/Application Insights resource (or equivalent logically-separated store), and each
client got its own Power BI workspace or Tableau site, with Azure AD identity mapping a viewer to
exactly one client's workspace. Where a shared semantic model was used for cross-cutting engineering
views, row-level security enforced the same boundary as a second, defense-in-depth control. The
reasoning for leaning on resource-level separation rather than trusting a filter alone: RLS is a
logical control that depends on every report and every future change applying it correctly forever; a
resource split is structural — there's no query path that can return both banks' rows even if a filter
rule is misconfigured or a new report is built incorrectly. For a regulated financial-services client,
that structural guarantee is worth the extra operational cost of maintaining two of everything.

---

### 21. Why does model risk monitoring matter more for a customer-facing production chatbot at a bank than it would for an internal prototype?

**Model answer:** Three things change once a system is customer-facing and in production at scale, and
each one raises the stakes of getting monitoring wrong. First, **blast radius** — a prototype's failure
affects the internal team that built it and can be pulled offline instantly; a production chatbot's
failure affects real account holders in real time, and by the time a bad pattern is noticed it may
already have reached thousands of interactions. Second, **regulatory exposure** — an internal prototype
isn't in scope for SR 11-7-style model risk governance because it isn't making or influencing decisions
that affect customers or the institution's regulatory standing; a production customer-facing assistant
at a bank is treated as an in-scope model requiring independent validation, ongoing monitoring, and a
documented audit trail, the same category of rigor as a credit-scoring model. Third, **reputational and
contractual exposure** — this assistant served two named clients, HSBC and Bank of America, so a
monitoring failure isn't just "our product had a bug," it's "a vendor failed to catch a problem in a
system serving one of the world's largest banks' customers," which is the kind of failure that ends
engagements, not just tickets. That combination — real customers, a regulator who explicitly expects
this category of oversight, and a named client relationship on the line — is why the monitoring layer
here was scoped as a compliance requirement from day one, not an engineering nice-to-have added after
launch.

---

### 22. How would a bank's model-risk officer actually use this dashboard day to day, and what would make them escalate to you?

**Model answer:** Day to day, it's mostly a quick check of the executive/trend view — is faithfulness,
hallucination rate, and harmfulness rate flat or trending in the right direction versus the last
30 days, and does anything correlate with a deployment marker from a recent prompt or retrieval change.
That's a five-minute glance most days, not a deep investigation. Weekly, they'd likely go a layer
deeper into the segment/breakdown view — checking whether any specific topic or intent category is
quietly underperforming even though the aggregate numbers look fine, since that's the pattern aggregate
metrics hide best. What would make them escalate to me directly, rather than just noting it for the
weekly review: a critical-tier alert firing (a bad-actor flag rate spike, or a harmful response that
reached a live user unfiltered), a sustained divergence between automated metrics and real user
signals (faithfulness scores look fine but thumbs-down or retry rate is climbing on a topic — that
mismatch means the metrics are missing something), or — specific to this being a two-client platform —
any sign, even a suspected one, that their dashboard might be showing data that isn't theirs. That last
category would get an immediate escalation regardless of how minor it looked, because for a named
banking client, a cross-tenant data question isn't something that waits for the weekly review; it's
treated as a potential incident until proven otherwise.
