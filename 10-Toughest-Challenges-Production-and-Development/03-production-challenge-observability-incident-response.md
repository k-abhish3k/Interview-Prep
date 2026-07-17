# 03 — Production Challenge: Observability & Incident Response

## Why "standard APM" isn't enough for a GenAI system

Traditional application performance monitoring answers questions like "is the service up, and how
fast is it responding?" — 5xx rates, latency percentiles, CPU/memory, queue depth. Every production
system in this curriculum needs that layer (Azure Monitor/Application Insights for the Capco
banking stack, CloudWatch for the Indegene pharma stack on ECS Fargate/Sagemaker/Lambda), and it's
necessary but nowhere near sufficient. A GenAI service can be **100% "healthy" by every standard
APM signal — no errors, normal latency, normal CPU — while quietly producing wrong, hallucinated,
or off-policy answers.** That's a fundamentally different failure mode: the system isn't broken,
it's *wrong*, and nothing in a standard dashboard says so. This is the chapter for an interviewer
who wants to know you understand production ML/LLM operations specifically, not just "can you run
a service."

## What LLM-specific observability actually has to catch

| Standard APM signal | LLM/ML-specific signal it misses |
|---|---|
| HTTP 5xx rate | Hallucination rate (the model answered confidently and wrong) |
| P50/P95/P99 latency | Retrieval quality drop (RAG pulling irrelevant context, so the model is grounded on the wrong thing) |
| CPU / memory | Groundedness/faithfulness score drift (answer no longer traceable to source documents) |
| Uptime / availability | Output-distribution drift (answers becoming shorter, more repetitive, or off-format after an upstream model version change) |
| Request volume | User behavior signals — thumbs-down rate, immediate re-ask/rephrase rate, session abandonment — that indicate quality problems users are silently absorbing |

Course 2 (Model Risk Monitoring) builds exactly the harness this chapter depends on: LLM and
non-LLM metrics (accuracy, completeness, hallucination, robustness, latency, harmfulness,
bad-actor), fed by a mix of cheap automated checks and LLM-as-judge scoring, plus user feedback
signals (thumbs up/down, reliance, engagement). In production, that harness isn't just an
offline evaluation tool — it's the **quality observability layer** that has to run continuously (or
on a representative sample of live traffic) and feed alerts the same way a latency dashboard does.

## On-call and incident response for a quality regression

A quality regression is a different kind of incident from a service outage, and the response
process has to account for that:

1. **Detection** — either an automated alert from the quality-monitoring pipeline (e.g.,
   hallucination-rate or groundedness-score crossing a threshold over a rolling window), a spike in
   user thumbs-down/re-ask rate, or — the worst-case path — a client escalation after a customer
   noticed a bad answer before the monitoring did.
2. **Triage** — is this a genuine model/prompt/data regression, or noise (a sampling artifact, a
   single bad batch of evaluation examples, a known-benign edge case)? Because LLM outputs are
   probabilistic, this triage step requires more judgment than "is the error rate above zero" — it's
   closer to statistical process control than to classic bug triage.
3. **Containment** — the standard levers are: **roll back** the last prompt/model/config change if
   one occurred recently (the most common root cause of a sudden regression), **pin** to a known-good
   model deployment version if the regression tracks an upstream provider model update, or add a
   **stricter output filter/guardrail** as a stopgap while the root cause is investigated.
4. **Root-cause and fix** — often requires re-running the golden evaluation set (course 6) against
   the suspected change to reproduce the regression offline, where it can be iterated on safely.
5. **Postmortem and monitoring update** — the fix isn't complete until the monitoring pipeline is
   updated to catch this specific failure mode earlier next time; a quality regression that reaches
   production without being caught by monitoring is itself a finding, not just the underlying bug.

## Rollback strategy for a bad prompt/model/config deploy

Because prompt, model-deployment, and config changes ship far more often than full application
code changes for a mature GenAI service, a fast, low-risk rollback path for *those* specifically is
worth designing on purpose rather than reusing the application's general CI/CD rollback:

- **Version every prompt and config as a first-class artifact** (not embedded as a string literal
  in application code) so a rollback is a config change, not a redeploy.
- **Deployment slots (Azure App Service) or blue/green task sets (ECS)** let a new
  prompt/model/config version be validated against a slice of traffic or a staging slot before a
  full swap, and give an instant swap-back if the quality-monitoring pipeline flags a regression
  shortly after go-live.
- **Keep the previous N versions of any prompt/model config addressable**, so "roll back to
  yesterday's known-good prompt" is a one-line operation, not an investigation.

## Illustrative incident: a silent quality regression caught before the customer noticed

> **Illustrative scenario, not a verified incident record.** Built from the Course 2 monitoring
> harness and the Capco/Azure production stack described above.

**Situation.** A prompt-engineering change was deployed to a banking client's AI Assistant (course
1) intended to make responses more concise, based on internal feedback that answers were running
too long. The change passed the standard pre-deploy checks — the service stayed healthy on every
standard APM signal (latency, error rate, uptime) after the deploy.

**Task.** As the engineer who also owned the Model Risk Monitoring pipeline (course 2) for this
assistant, I was responsible for making sure a prompt change like this didn't quietly reduce answer
quality, and for catching it before a client-side risk/compliance reviewer or an end user did.

**Action.** The monitoring pipeline's rolling groundedness/completeness scores, computed on a
sampled slice of live traffic via the LLM-as-judge evaluation harness, showed a step-change drop
starting within the hour after the deploy — the more concise responses were, on a meaningful
fraction of queries, dropping supporting detail that had previously anchored the answer to a
specific source document, i.e., trading completeness for brevity in a way that increased
hallucination risk on multi-part questions. I flagged the anomaly against the deploy timeline
(the monitoring dashboard tracks deploy events alongside the metric series specifically to make
this correlation fast), confirmed it against a manual review of a sample of the flagged responses,
and rolled the prompt back to the previous known-good version via the versioned prompt config
rather than waiting for a full root-cause writeup.

**Result.** *(Illustrative)* The rollback happened within roughly two hours of the original
deploy, before any client escalation or user-visible complaint reached the support queue. The
concise-response goal was later re-implemented as a smaller, more targeted prompt change that kept
supporting-detail sentences intact, validated against the golden evaluation set (course 6) before
redeploying, and the groundedness metric was added to the standard pre-deploy checklist gate for
any future prompt change on this assistant.

## Talking points this chapter sets up

- The core distinction: standard APM tells you the service is *up*; LLM-specific observability
  tells you the service is *right* — and a GenAI production system needs both.
- Continuous/sampled quality monitoring (course 2's harness, running in production) as the
  detection mechanism for the kind of incident classic monitoring structurally cannot see.
- A rollback strategy purpose-built for prompt/model/config changes (versioned artifacts,
  slots/blue-green), because those change far more often than application code.
- Treating "monitoring didn't catch it in time" as part of the incident's root cause, not a
  separate issue — closing that gap is part of the fix.
