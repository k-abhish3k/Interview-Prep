# 02 — Capco: Model Risk Monitoring for AI Assistant

## Resume bullet this course is built from

> "Model Risk Monitoring Model to evaluate AI Assistant: Integrated Model Monitoring platform to
> assess AI Assistant performance and moderation. Evaluated responses using LLM and non-LLM metrics
> including accuracy, completeness, hallucination, robustness, efficiency, latency, harmfulness,
> bad-actor. Incorporated user feedback, reliance, and engagement for moderation insights."

This course is the companion to [`01-Capco-AI-Chatbot-Assistant`](../01-Capco-AI-Chatbot-Assistant/00-README.md).
That course covers *building* an AI Assistant. This one covers *watching* it once it's live.

In plain terms: this course is about Model Risk Management (MRM). MRM is the discipline of watching a
model in production to make sure it stays accurate and safe. This course applies that discipline to a
generative AI system, and covers the evaluation harness (the code and tools that score the assistant's
answers) that a Consultant-II would plausibly design and build around it.

## Why financial-services firms need "model risk management"

Capco is a management and technology consultancy. It works almost exclusively with banks, insurers, and
other regulated financial institutions.

In that world, deploying a model — any model, not just an LLM (Large Language Model) — is not just an
engineering decision. It's a **governance event**. That means someone in a risk or compliance role has
to formally sign off on it, and keep checking on it afterward.

Here's why. In 2011, U.S. banking regulators (the Federal Reserve and the OCC) issued joint guidance
called **SR 11-7** ("Supervisory Guidance on Model Risk Management"). Since then, banks operating in or
with exposure to U.S. regulators have been expected to treat models as their own risk category — the
same way they'd treat credit risk or operational risk. Similar rules exist elsewhere: the UK's PRA
SS1/23, and emerging EU AI regulation.

The core idea behind all of these rules is simple to state, and hard to actually execute:

- Every model can be wrong. A wrong model making decisions at scale can cause real financial and
  reputational harm.
- So every model needs three things: **independent validation** before and during production use,
  **ongoing monitoring** of how it behaves, and **documented evidence** that someone is watching for
  drift, bias, and failure.
- The definition of "model" is broad. Most banks now treat generative AI systems — chatbots, copilots,
  summarizers — as in-scope models. That means they need the same rigor as, say, a credit scoring model.

This is the regulatory backdrop that makes a bullet like "Model Risk Monitoring Model to evaluate AI
Assistant" make sense as a real, fundable project inside a bank-facing engagement. It's very unlikely a
bank would let an LLM-based assistant go live for real users without *some* evaluation and monitoring
layer that a risk or compliance function can point to. Someone has to build that layer — that's the
plausible scope of this project.

This isn't a hypothetical backdrop, either. The named clients here are **HSBC** and **Bank of
America** — both subject to SR 11-7 and/or its UK PRA (SS1/23) equivalent. This is exactly the kind of
ongoing governance evidence a vendor like Capco is expected to produce, before and continuously after, a
customer-facing chatbot goes live for real account holders.

## Client & Production Deployment

This platform was built at Capco to monitor the **AI Chatbot Assistant from Course 01**, for two named
banking clients: **HSBC** and **Bank of America**.

That assistant was not a proof-of-concept. It was **customer-facing and running in production**,
handling real daily traffic. It ran on **Azure App Service**, fronted by Azure Front Door/Application
Gateway WAF (a web firewall), VNet-integrated with Private Endpoints, secured with Azure AD, and backed
by Azure OpenAI. **Azure Monitor / Application Insights** was the natural home for the telemetry (the
raw usage and performance data) that this project's evaluation harness and metrics store were built on
top of.

Because one platform served two competing banking clients, **per-client metric isolation was a hard
requirement, not a nice-to-have**. HSBC's conversation data, scores, and dashboards must never be
visible in Bank of America's environment, and vice versa. A bank's model-risk officer should only ever
see their own institution's data. Anything else is itself a reportable incident — not a UX bug.

This reinforces the SR 11-7 framing above: banking regulators expect strict per-entity governance. A
monitoring vendor that let two clients' data mix would fail that expectation immediately, no matter how
good the metrics themselves were. Chapter 05 covers the dashboard-level mechanics (row-level security
vs. separate workspaces) used to enforce this.

## The candidate's likely role

Course 01 built the AI Assistant. The natural next engagement is: *"Now prove it's safe and good enough
to keep running, and give the bank a way to keep proving that over time."*

That breaks down into work a backend GenAI developer would actually own:

- **Design a set of evaluation metrics** that map to what a risk committee cares about: is the
  assistant *accurate*, *complete*, prone to *hallucination* (making things up), *robust* to rephrased
  or adversarial input, fast enough (*latency*/*efficiency*), and safe from *harmful* or *bad-actor*
  misuse.
- **Wire up an evaluation harness** that scores assistant responses on those metrics. This is a mix of
  cheap non-LLM checks (regex/keyword/statistical rules) and more expensive LLM-as-judge checks (using
  a second AI model to grade the first one's answer), run either synchronously on a sample of live
  traffic or as a batch/offline regression suite. See the
  [Model Monitoring integration pattern in Azure ML](https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-monitoring)
  for the kind of platform this likely plugged into or resembled.
- **Feed those scores into a dashboard.** Combine the metric scores with **user feedback** (thumbs
  up/down), **reliance** (did the user act on the answer, or immediately reject/retry it), and
  **engagement** (session length, repeat usage) to produce moderation insights a risk team can actually
  use — a Power BI or Tableau layer sitting on top of the metrics store.
- **Use explainability tooling** (SHAP, LIME, Shapash — covered in Chapter 03) to make individual
  scoring decisions auditable. Not just "the model said this response was 62% likely to hallucinate,"
  but *why*.

Framed this way, the project isn't "build a chatbot" — that's Course 01. It's "build the control tower
that lets a bank trust the chatbot." That distinction is worth making explicit in interviews.

## Monitoring pipeline (conceptual)

```mermaid
flowchart LR
    U[End User<br/>HSBC or BofA channel] --> A[AI Assistant<br/>Azure App Service - production<br/>Course 01]
    A -->|response| E{Metric Evaluators}
    E --> M1[Non-LLM metrics<br/>accuracy, robustness,<br/>latency, PII/toxicity]
    E --> M2[LLM-as-judge metrics<br/>completeness, hallucination,<br/>harmfulness]
    E --> M3[Ragas RAG metrics<br/>faithfulness, relevancy,<br/>context precision/recall]
    M1 --> S[(Metrics Store<br/>Azure Monitor / App Insights,<br/>partitioned per client)]
    M2 --> S
    M3 --> S
    S --> D1[HSBC Dashboard<br/>Power BI / Tableau]
    S --> D2[Bank of America Dashboard<br/>Power BI / Tableau]
    S --> AL[Alerting<br/>threshold breaches, scoped per client]
    U -->|thumbs up/down,<br/>reliance, engagement| FB[Feedback Signals]
    FB --> S
    D1 --> RC1[HSBC Model Risk Team]
    D2 --> RC2[BofA Model Risk Team]
    AL --> RC1
    AL --> RC2
    RC1 -->|tuning, guardrail updates,<br/>retraining triggers| A
    RC2 -->|tuning, guardrail updates,<br/>retraining triggers| A
```

Plain-text version, if diagram rendering isn't available:

```
End User (HSBC or BofA channel) -> AI Assistant (Azure App Service, production, Course 01) -> [response] -> Metric Evaluators
                                             |-- Non-LLM metrics (accuracy, robustness, latency, harmfulness rules)
                                             |-- LLM-as-judge metrics (completeness, hallucination, harmfulness)
                                             `-- Ragas RAG metrics (faithfulness, relevancy, context precision/recall)
Metric Evaluators -> Metrics Store (Azure Monitor / App Insights, partitioned per client: HSBC vs. BofA)
Metrics Store -> HSBC Dashboard (Power BI / Tableau)        -> HSBC Model Risk Team
Metrics Store -> BofA Dashboard (Power BI / Tableau)        -> BofA Model Risk Team
Metrics Store -> Alerting (threshold breaches, scoped per client) -> HSBC / BofA Model Risk Team (respectively)
End User -> feedback (thumbs up/down, reliance, engagement) -> Metrics Store
HSBC / BofA Model Risk Team -> tuning / guardrail updates / retraining triggers -> AI Assistant  (feedback loop closes, per client)
```

The key change from a single-tenant version of this diagram: the metrics store, dashboards, and
alerting are **partitioned per client from the point of ingestion onward**. There is no shared
"all clients" view anywhere in this pipeline. See Chapter 05 for how that partitioning is enforced
at the dashboard layer.

The important shape to internalize: this is a **closed loop**, not a one-way pipeline. Monitoring
that doesn't feed back into changes to the assistant (prompt tuning, retrieval fixes, guardrail
updates) is just a report nobody acts on. In an SR 11-7 context, "monitoring with no action loop" is
itself a documented finding an auditor will flag.

## How this course is organized

| File | Covers |
|---|---|
| `01-llm-evaluation-fundamentals.md` | Why evaluating generative text is hard; the metric taxonomy; LLM-as-judge vs non-LLM metrics |
| `02-ragas-and-rag-metrics.md` | Ragas framework: faithfulness, answer relevancy, context precision/recall |
| `03-explainable-ai-shap-lime-shapash.md` | SHAP, LIME, Shapash — explainability for audit and accuracy/completeness checks |
| `04-robustness-adversarial-and-safety-testing.md` | Perturbation testing, prompt injection / bad-actor testing, harmfulness classification, reliance/engagement signals |
| `05-building-a-monitoring-dashboard.md` | Turning per-response metrics into a Power BI/Tableau dashboard with alerting and feedback loops |
| `06-baseline-staleness-and-drift-lifecycle.md` | The "what happens when the assistant changes" gotcha: rolling-baseline staleness vs. genuine drift, and a proposed baseline-versioning fix tied to an assistant version tag |
| `07-production-resilience-and-operational-engineering.md` | Evaluator error-handling table, concurrency/race caveats, illustrative bug narratives, concrete timeout/retry/threshold values, and a named hardening gap |
| `99-Interview-QA.md` | Interview questions + model answers |
| `notebooks/` | Runnable from-scratch implementations of the ideas above |

## STAR summary

> **Situation** — Capco's clients, **HSBC** and **Bank of America**, had already deployed a generative
> AI Assistant (see Course 01) that was **customer-facing and live in production** on Azure App
> Service, handling real daily traffic. Because both clients are regulated financial institutions,
> their model risk / compliance functions required documented, ongoing evidence that the assistant's
> outputs were accurate, safe, and monitored in line with SR 11-7-style model risk frameworks — the
> assistant could not simply run unobserved in production, and metrics for the two banks could never
> be mixed.
>
> **Task** — As the GenAI backend developer, integrate the AI Assistant with a Model Monitoring
> platform and design the evaluation logic: define and implement metrics covering accuracy,
> completeness, hallucination, robustness, efficiency, latency, harmfulness, and bad-actor/adversarial
> resistance, and combine them with user feedback, reliance, and engagement signals to produce
> moderation insights the risk team could act on.
>
> **Action** — Built an evaluation harness that scored live and/or sampled assistant responses using
> both LLM-as-judge techniques (for semantic properties like completeness and hallucination) and
> cheaper non-LLM/statistical checks (for latency, efficiency, rule-based harm detection). Used a
> Ragas-style decomposition for RAG-specific quality signals (faithfulness, relevancy, context
> precision/recall), applied explainability tooling (SHAP/LIME/Shapash) so scoring decisions were
> auditable rather than opaque, and piped everything into a **per-client-partitioned** metrics store
> (built on Azure Monitor/Application Insights) feeding separate Power BI/Tableau dashboards with
> threshold-based alerting for HSBC and Bank of America, closing the loop with real user feedback
> (thumbs up/down, reliance, engagement).
>
> **Result (illustrative — replace with your real numbers)** — The monitoring layer surfaced an
> estimated **~85% of hallucinated or unsupported responses** before they reached end users
> unflagged, and cut the average time for the risk/moderation team to detect a metric-threshold
> breach from a manual weekly review down to **near real-time alerting**, giving the client the
> auditable evidence trail their model risk framework required. *(Replace the 85% figure, the
> baseline detection cadence, and any other numbers here with your actual measured results before
> using this in an interview — reviewers will probe for specifics.)*

## A note on accuracy of claims in this course

Capco's internal architecture for this project is not public, and this course does not claim to know
it. Wherever this course describes "how it was likely built," treat that as an informed, defensible
guess, based on three things: (a) the resume bullet's exact wording, (b) standard MRM/LLM-eval practice
in regulated industries, and (c) the tools listed on the candidate's skill list (Ragas, Shapash, LIME,
Power BI, Tableau, Azure ML).

In an interview, be honest that some details are reconstructed rather than verbatim recalled.
Interviewers respect "here's the architecture I'd defend" more than a suspiciously too-perfect memory of
a project from over a year ago.

Before naming HSBC or Bank of America out loud to an interviewer, check what your actual NDA/engagement
letter allows. See the confidentiality note in the root [`README.md`](../README.md) for the safe
"top-3 global bank" style phrasing to fall back on if needed.
