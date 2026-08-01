# 04 — Production Challenge: Cost & LLM Quota Management

## Why this is a distinct problem from scaling

Chapter 01 covers making sure the system *can* handle load. This chapter covers making sure it
handles that load **affordably and fairly**, which turns out to be a genuinely different set of
tradeoffs. A platform team supporting multiple enterprise clients on a shared platform — HSBC and
Bank of America on the Capco side, Eli Lilly and AstraZeneca on the Indegene side — has to answer
questions that don't come up for a single-tenant system: whose budget pays for what, how do you
stop one client's usage spike from degrading service or blowing the shared budget for everyone
else, and how do you keep steady-state cost proportional to actual value delivered rather than
just "however much compute we happened to leave running."

## The four cost/quota levers in play

### 1. Azure OpenAI token budget per client

Azure OpenAI is billed and quota-limited on tokens (input + output) per deployment, on top of the
TPM/RPM rate limits covered in chapter 01. On a platform serving two banking clients from a shared
engineering team, the practical questions are:
- **Is each client on its own Azure OpenAI deployment** (cleanest for both cost attribution and the
  tenant-isolation requirement from chapter 02), or a shared deployment with usage tracked by
  tagging/metadata? Per-client deployments cost more in provisioned-throughput terms but make cost
  attribution and quota fairness trivial — this is usually the right tradeoff for regulated clients
  who need clean cost and usage reporting anyway.
- **Prompt-length optimization** directly reduces token spend: trimming unnecessary system-prompt
  boilerplate, truncating retrieved context to what's actually relevant (a RAG pipeline that stuffs
  the full top-k retrieved chunks into every prompt regardless of relevance score is a common,
  avoidable source of token waste), and choosing a smaller/cheaper model tier for sub-tasks that
  don't need the flagship model's capability (e.g., a lightweight classification or extraction step
  ahead of the main generation call).

### 2. Sagemaker endpoint cost vs. scale-to-zero tradeoffs

A Sagemaker real-time endpoint bills for provisioned instance-hours whether or not it's actively
serving traffic — the opposite failure mode from Azure OpenAI's pay-per-token model. The tradeoffs:
- **Always-on provisioned endpoints** give predictable low latency (no cold start) but cost the same
  at 2am with zero traffic as at 2pm with peak traffic — wasteful for a workload with a clear daily
  traffic curve (see `notebooks/02_autoscaling_capacity_planning_simulator.ipynb` for a runnable
  model of exactly this daily curve).
- **Sagemaker Serverless Inference or scheduled scale-to-zero/scale-up** trades cold-start latency
  (real, often seconds, sometimes tens of seconds for larger models) for meaningfully lower cost
  during low-traffic windows — a legitimate choice for an internal or asynchronous-batch workload,
  a much harder sell for a customer-facing endpoint with a strict latency SLA.
- **Multi-model endpoints** (hosting several models behind one endpoint, loaded on demand) can
  amortize fixed instance cost across several lower-traffic models — useful when a client has
  several narrow classification/detection models (e.g., the superscript-detection and
  chart-detection models from courses 7 and 10) that individually don't justify a dedicated
  always-on endpoint.

### 3. Caching repeated queries

A meaningful share of production LLM traffic in an enterprise-knowledge-base or FAQ-style assistant
is exact or near-duplicate questions. A response cache (keyed on normalized query text, or on
embedding similarity for near-duplicates) can serve a large fraction of repeat queries without a
model call at all — the highest-leverage cost lever available, since it reduces both token spend
and load on the rate-limited inference tier simultaneously. The tradeoff is staleness: a cached
answer needs an invalidation strategy tied to underlying source-document updates, or it becomes a
correctness risk, not just a cost optimization.

### 4. Prompt-length and pipeline-shape optimization

Beyond trimming an individual prompt, the pipeline shape itself is a cost lever: does every request
need a full retrieval + rerank + generation chain, or can a cheap upfront classifier route simple
queries to a shorter, cheaper path? This is the same instinct as tiered-model routing, applied to
the pipeline rather than just the model choice.

## Illustrative incident: a client's usage pattern blows through a quota/cost budget

> **Illustrative scenario, not a verified incident record.** Constructed from the Azure
> OpenAI/Sagemaker cost model above and the realistic shape of enterprise usage patterns.

**Situation.** A pharma client (illustrative: AstraZeneca-class) had a GenAI content-generation
feature (in the shape of course 11's regulatory-document generation platform) provisioned with a
monthly token/cost budget sized against its pilot-phase usage pattern — a modest, steady rate of
document-generation requests from a small reviewer team. After the feature was rolled out more
broadly inside the client organization following a successful pilot, usage grew several-fold within
weeks, driven by a much larger population of end users who had not been part of the original
capacity plan.

**Task.** As the engineer responsible for the platform's cost and quota management across clients,
I needed to understand why the budget was being consumed far faster than projected, prevent it from
degrading service for this client or (given shared infrastructure components) any other client, and
propose a sustainable fix rather than just requesting an emergency budget increase.

**Action.** I pulled per-client token usage from the Azure OpenAI/Sagemaker usage metrics (tagged
per client per chapter 02's isolation model, which made this analysis straightforward rather than a
forensic exercise) and found the growth was concentrated in a specific query pattern: end users were
frequently re-generating near-identical documents with minor parameter changes, each a full,
expensive generation call with no reuse of the unchanged portions. I implemented a two-part fix:
a response cache keyed on the normalized generation parameters for the sections of a document that
hadn't changed between re-generation requests, and a client-visible usage dashboard so the client's
own team could see their consumption trend before it became a surprise on either side. I also
proposed — and the account team agreed — a revised budget tier that reflected the feature's actual
broader-rollout usage rather than its pilot-phase usage, so the next scale-up wouldn't repeat the
same surprise.

**Result.** *(Illustrative)* The caching fix reduced token spend for the flagged usage pattern by a
significant fraction (illustrative: roughly 30–40% of the growth was addressable via caching alone),
and the revised budget tier combined with the usage dashboard meant the client's own stakeholders
had visibility into consumption trends going forward rather than learning about a budget event after
the fact. The per-client usage tagging and dashboard were later adopted as a standard rollout
checklist item for any client moving from pilot to broader deployment.

## Talking points this chapter sets up

- Cost/quota management as a fairness and reliability problem on a shared multi-client platform,
  not just a finance conversation.
- The opposite cost shapes of pay-per-token (Azure OpenAI) versus pay-per-instance-hour (Sagemaker
  endpoints), and how that shapes the scale-to-zero decision differently for each.
- Caching as the highest-leverage lever because it reduces cost and rate-limit pressure together.
- Treating a pilot-to-broad-rollout usage jump as a predictable capacity-planning event, not a
  surprise, once you've seen it happen once.
