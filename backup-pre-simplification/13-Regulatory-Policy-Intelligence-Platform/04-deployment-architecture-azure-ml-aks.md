# 04 — Deployment Architecture: Azure ML Managed Online Endpoints vs. AKS + vLLM/TGI

## Why this chapter exists

Chapter 2 established *why* this project self-hosts Mixtral. This chapter is *how*: the concrete Azure
serving architecture, the real tradeoff between the two credible hosting options, GPU sizing
considerations, autoscaling, and quantization as a cost/latency lever. This is the chapter to reach for
when an interviewer asks "OK, you decided to self-host — walk me through what that actually looks
like."

## Two credible hosting options, and the honest tradeoff between them

**Option A: Azure ML managed online endpoints.** Azure Machine Learning's managed online endpoints let
you deploy a model (including a custom container running Mixtral behind an inference server) behind a
managed HTTPS endpoint, with Azure handling the underlying compute provisioning, built-in autoscaling
rules, traffic-splitting for blue/green or canary rollouts, and integrated logging/monitoring. The
operational burden on the client's platform team is comparatively low — you supply a container image
and a compute SKU, Azure ML handles the rest of the endpoint lifecycle.

**Option B: AKS (Azure Kubernetes Service) with a dedicated serving framework** — **vLLM** or **Text
Generation Inference (TGI)** — running as a workload on GPU-backed AKS node pools. This gives full
control over the serving stack: how requests are batched, how the KV cache is managed, how multiple
model replicas and multiple LoRA adapters (Chapter 5) are scheduled across nodes, and exactly how
autoscaling triggers are wired (see below). The operational burden is meaningfully higher — the
platform team owns the Kubernetes cluster, the serving framework's configuration and upgrades, and the
autoscaling/observability wiring that Azure ML would otherwise provide out of the box.

**The tradeoff, stated as a decision rule rather than a preference:**

| Traffic profile | Favors |
|---|---|
| Moderate, relatively predictable volume; team wants to minimize ongoing ops burden; fast time-to-production matters more than squeezing out maximum throughput | **Azure ML managed online endpoint** |
| Very high, sustained throughput where request-batching efficiency materially changes unit economics; need for advanced serving features (continuous/dynamic batching tuned precisely, multi-adapter serving at scale, fine-grained control over scheduling) | **AKS + vLLM/TGI** |
| Team already operates AKS clusters for other workloads and has in-house Kubernetes expertise | **AKS + vLLM/TGI** — the marginal ops cost of one more workload on an already-operated cluster is lower than standing up net-new Kubernetes expertise |
| Team has no existing Kubernetes footprint and this is the first GPU-serving workload the platform will run | **Azure ML managed online endpoint** — avoid taking on Kubernetes operational risk and self-hosting operational risk simultaneously |

For this platform's actual profile — sustained, high-volume batch-and-near-real-time document
processing, but not latency-critical interactive chat traffic — a defensible starting position is
**Azure ML managed online endpoints for the initial production rollout**, explicitly evaluated against
an AKS + vLLM migration path once real production volume and throughput requirements are measured
rather than estimated. That sequencing — start with the lower-ops-burden option, graduate to the
higher-control option only once volume justifies the added complexity — is itself a defensible
architectural position to state directly if asked "why not just go straight to AKS for maximum
control."

## GPU instance sizing considerations

Sizing is driven by three things that all compete for the same GPU memory budget: **the model's own
weights**, **the KV cache** used during generation, and **headroom for concurrent requests**.

- **Weights footprint.** Mixtral 8x7B's ~47B total parameters (Chapter 1) need to be held in GPU memory
  at whatever precision is being served. At full (FP16/BF16) precision, that's roughly 2 bytes per
  parameter — meaningfully more memory than a single consumer or mid-tier GPU offers, which is why
  quantization (below) is a live lever here, not a hypothetical one.
- **KV cache.** As generation proceeds token by token, the attention mechanism's key/value cache grows
  with sequence length and batch size (partially mitigated by GQA, Chapter 1, which reduces the KV
  cache's size relative to standard multi-head attention, but doesn't eliminate the scaling). Long
  regulatory documents mean long input sequences, which means a non-trivial KV cache footprint even
  before generation begins.
- **Concurrency headroom.** Multiple in-flight requests each need their own KV cache allocation; sizing
  for realistic concurrent load (not just a single request's footprint) is what actually determines
  throughput under production traffic.

The practical sizing question is described generically here deliberately, rather than committing to a
specific GPU SKU name that risks going stale: Azure's GPU-backed VM families intended for inference
workloads (the NC-series and ND-series families, broadly) offer a range of GPU memory capacities, and
the right choice is whichever tier comfortably fits the weights-plus-KV-cache-plus-concurrency-headroom
budget above for the target request volume and sequence length — sized against measured production
request patterns, not guessed once at design time and left alone, the same principle course 11's
LLaMA 3 deployment chapter applies to its own Sagemaker instance sizing.

## Autoscaling triggers

Autoscaling a GPU-backed LLM serving layer should trigger on the metric that actually reflects the
bottleneck, which for a Mixtral serving layer is **GPU utilization and/or queued-request depth**, not
generic CPU utilization — the CPU is rarely the constrained resource in a GPU-serving workload, and
scaling on it can miss the actual bottleneck entirely (Chapter 8 covers a concrete bug narrative built
around exactly this mistake). Reasonable trigger signals, in priority order:

1. **GPU utilization** — scale out when average GPU utilization across current replicas sustains above
   a threshold (e.g., a sustained high-utilization window, not a single instantaneous spike, to avoid
   flapping).
2. **Request queue depth / time-in-queue** — scale out when requests are waiting materially longer than
   the endpoint's target latency SLO before being picked up for inference, which is a more direct signal
   of capacity exhaustion than utilization alone.
3. **Scale-in on the inverse of both**, with a longer cooldown on scale-in than scale-out — GPU instances
   are expensive and slow to provision (driver/model load time is not instant), so it's better to be
   slightly slow to release capacity than to thrash between scaling up and down under bursty traffic.

## Quantization as a cost/latency lever

**Quantization** reduces the numeric precision used to store and compute with the model's weights — for
example, running Mixtral in an 8-bit or 4-bit quantized form instead of full FP16/BF16 precision. The
practical effect is a roughly proportional reduction in memory footprint (an 8-bit quantized model
needs roughly half the memory of the same model at 16-bit precision; 4-bit roughly a quarter), which
directly translates into either fitting the model on a smaller/cheaper GPU instance tier, or fitting
more concurrent request capacity (larger KV cache headroom) on the same instance tier — and often
faster inference too, since moving less data through memory bandwidth is frequently the bottleneck in
practice, not raw compute.

The tradeoff is a **quality cost**, and it's a real one, not a rounding error: aggressive quantization
can measurably degrade output quality, and the degradation is not necessarily uniform across task types
— it can show up disproportionately on the harder, more nuanced parts of a task (subtle classification
distinctions, precise field extraction) while barely affecting the easier, more templated parts.
Chapter 8 walks through a concrete illustrative bug story built around exactly this: a quantization
config that passed general-purpose evaluation but silently degraded accuracy on one specific document
category, caught only by the domain-specific evaluation set from Chapter 5. The right way to reason
about it, and the way to state it in an interview: quantization level is a tunable point on a
cost/latency/quality curve, not a single "on or off" switch, and the only responsible way to choose a
point on that curve is to measure quality against the actual domain evaluation set at each candidate
quantization level, not just trust a published general-benchmark quality delta. Notebook
`03_quantization_cost_latency_tradeoff.ipynb` models this tradeoff conceptually — comparing a synthetic
"full precision" and "quantized" configuration's memory footprint and relative throughput, with a
clearly-labeled illustrative quality-degradation estimate.

## Tying It Back

The deployment story for this platform is: start with Azure ML managed online endpoints to minimize
operational risk while proving out real production volume, keep AKS + vLLM/TGI as an explicitly
evaluated higher-throughput migration path once volume justifies the added operational complexity, size
GPU instances against weights-plus-KV-cache-plus-concurrency-headroom measured from real traffic rather
than guessed once, trigger autoscaling on GPU utilization and queue depth rather than a generic CPU
metric that would miss the real bottleneck, and treat quantization as a deliberately-measured point on
a cost/latency/quality curve rather than a default cost-cutting switch flipped without checking the
domain evaluation set from Chapter 5.
