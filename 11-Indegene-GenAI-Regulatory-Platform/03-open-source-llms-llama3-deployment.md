# 03 — Open-Source LLMs: LLaMA 3 Deployment & Tradeoffs

## Why this chapter matters

The resume bullet names LLaMA 3 specifically, not "an LLM" generically — and in an interview, "why
did you use an open-weight model instead of just calling GPT-4/Azure OpenAI" is one of the first
architecture questions you should expect. This chapter builds the case from first principles: what
"open-weight" actually buys you, what it costs you in operational responsibility, and how those
tradeoffs land specifically for a platform generating patient-facing and regulatory clinical content.

## Open-weight vs. closed API models

A **closed API model** (Azure OpenAI's GPT-4 class, AWS Bedrock's Claude/Titan) is called over the
network — you send a prompt, the provider runs it on their infrastructure, and you get a completion
back. You never see or control the weights. An **open-weight model** (LLaMA 3, Mistral, and similar)
ships its trained parameters to you; you run it (or pay someone to run it for you) on infrastructure
you control, and you can inspect, quantize, and fine-tune it directly.

Neither is universally "better" — they trade off along a few axes that matter differently depending
on the workload:

| Axis | Closed API | Open-weight (self-hosted) |
|---|---|---|
| **Data residency / privacy** | Prompt content leaves your environment to the provider | Data never leaves infrastructure you control |
| **Customization** | Limited to prompting, few-shot, provider-hosted fine-tuning APIs (if offered) | Full fine-tuning/PEFT access to weights, full control over training data |
| **Cost at scale** | Per-token pricing, scales linearly with volume, no infra to manage | Upfront/ongoing infra cost (GPUs), but marginal cost per call drops sharply at high, steady volume |
| **Operational responsibility** | Provider handles scaling, uptime, model updates | You own serving infrastructure, scaling, monitoring, patching |
| **Latest-model access** | Provider ships frontier model improvements | Open-weight models typically lag frontier closed models in raw capability |

## Why an open-weight model is a defensible choice here

**Data residency and privacy for sensitive clinical data.** Clinical trial Protocols routinely
contain unpublished trial design details — proprietary endpoints, dosing strategies, statistical
plans — that sponsors treat as confidential IP, on top of the direct patient-safety sensitivity of
anything that will become participant-facing consent language. Being able to run inference (and, for
fine-tuning, training) entirely inside infrastructure the organization controls, without sending that
text to a third-party API, is a strong argument for an open-weight model in this domain — independent
of any specific regulation, sponsors and Institutional Review Boards care a great deal about exactly
where their unpublished protocol content goes.

**Cost at scale.** A platform processing many Protocol documents, each broken into many sections,
each of which needs generation calls for several modules (ICF, PLPS, SOC, comparison) plus retries
and reviewer-requested regenerations, produces a lot of inference volume. Per-token API pricing scales
linearly with that volume; self-hosted inference has a largely fixed infrastructure cost that
amortizes better as volume grows. The crossover point where self-hosting becomes cheaper than a
closed API depends on actual volume and GPU pricing, but for a platform running as continuous
production infrastructure rather than sporadic calls, it's a calculation worth doing explicitly
rather than defaulting to "the API is always simpler."

**Customizability via fine-tuning.** Chapter 02 covered why LoRA/QLoRA fine-tuning matters for
getting reliable, format-specific behavior out of the model for ICF/PLPS/SOC generation. Open-weight
models give direct access to the weights needed for that; a closed API constrains you to whatever
fine-tuning surface the provider chooses to expose (if any), on their terms and their infrastructure.

## Self-hosting and deployment considerations

Choosing an open-weight model is the easy part; running it reliably in production is where the real
engineering is. Three concerns dominate:

### Quantization

LLaMA 3's larger variants (8B, 70B parameters) require substantial GPU memory to run at full
(FP16/BF16) precision — a 70B model at 16-bit precision needs roughly 140GB just for weights, before
accounting for the KV cache used during generation. **Quantization** reduces the numeric precision
used to store weights (e.g., to 8-bit or 4-bit integers) with a controlled accuracy tradeoff, cutting
memory footprint roughly in proportion to the bit-width reduction. For this platform, the practical
question is where on that accuracy/cost curve to sit: a smaller or more aggressively quantized model
serving routine, well-templated sections (SOC entries, for instance, which are close to fill-in-the-
blank structured records) versus a less-quantized, larger model reserved for the more nuanced
patient-facing narrative generation (ICF, PLPS), where subtlety of language matters more.

### Serving frameworks

Running a raw model checkpoint naively (a simple forward-pass loop) leaves significant throughput on
the table. Production serving frameworks (e.g., vLLM, TGI/Text Generation Inference, or a Sagemaker/
Bedrock-managed serving layer if choosing a managed-but-still-open-weight route) add:

- **Continuous/dynamic batching** — packing multiple concurrent requests together efficiently instead
  of processing one at a time, which matters a lot when many section-generation calls are in flight
  across a Protocol at once.
- **KV-cache management** — reusing attention key/value computations across the autoregressive
  generation loop instead of recomputing them, which is the single biggest inference-speed lever for
  transformer decoding.
- **Multi-adapter serving** — some serving frameworks support loading a shared frozen base model once
  and swapping/serving multiple LoRA adapters on top of it concurrently, which lines up directly with
  the "one base model, per-module LoRA adapters" design from Chapter 02.

### Autoscaling and monitoring

Just like the AWS-based deployments elsewhere on the candidate's resume (Sagemaker endpoints with
auto-scaling, CloudWatch-driven health checks — see course 09), a self-hosted LLM serving layer still
needs request-volume-driven autoscaling, latency/error monitoring, and a rollback path when a new
fine-tuned adapter regresses quality — the operational discipline doesn't disappear just because the
model isn't a third-party API; if anything, more of it becomes your responsibility.

## Tradeoffs vs. Azure OpenAI / AWS Bedrock

It's worth being explicit in an interview that this isn't a one-time, permanent decision — it's a
workload-by-workload judgment call, and a realistic platform likely uses **both**:

- **Self-hosted LLaMA 3** for the core, high-volume, domain-fine-tuned generation work (ICF/PLPS/SOC
  section drafting) where data residency, cost at scale, and deep customization via LoRA matter most.
- **A managed API (Azure OpenAI or AWS Bedrock)** as a viable alternate backend for lower-sensitivity,
  lower-volume, or exploratory tasks — e.g., early prototyping of a new section type before investing
  in fine-tuning data collection, or as a documented fallback if self-hosted capacity is constrained.
  This is exactly the kind of "AWS Bedrock as alt LLM backend" framing noted in this course's entry
  in the curriculum's skills matrix — worth naming as a considered alternative even where LLaMA 3 was
  the primary choice, since interviewers respect "I evaluated X and chose Y because..." over "I only
  ever considered one option."

The deciding factors, roughly in priority order for this domain: (1) can sensitive protocol/patient
content leave the organization's controlled environment — usually no; (2) does the workload need deep
fine-tuning control — for ICF/PLPS/SOC generation, yes; (3) does the expected volume justify the fixed
cost of self-hosted infrastructure — for a platform running continuously across many trials, likely
yes. Those three answers point toward self-hosted LLaMA 3 as the primary generation backend, with a
managed API kept available as a secondary/fallback option.

## Production deployment: a Sagemaker real-time endpoint, not a demo

This wasn't a research exercise — the fine-tuned LLaMA 3 model served two real pharma clients, **Eli
Lilly** and **AstraZeneca**, whose regulatory-affairs teams called it every working day to draft ICF,
PLPS, and SOC content. That changes "how do you serve an open-weight model" from an academic question
into an operational one, and it's worth being able to describe the production shape concretely.

- **Sagemaker real-time inference endpoint, not Bedrock, not a raw EC2 box you SSH into.** The
  fine-tuned/instruction-tuned LLaMA 3 model was hosted behind a **Sagemaker real-time endpoint** —
  Sagemaker gives you managed model hosting (versioned model artifacts, endpoint configs, built-in
  auto-scaling and health checks) while still running a model you control the weights of, which is the
  middle ground between "roll your own EC2 + vLLM cluster" (maximum control, maximum operational
  burden) and "call Bedrock" (zero hosting burden, but you're back to a third-party-hosted model,
  which was the thing being avoided in the first place — see below). For a platform that needed both
  self-hosting and production-grade reliability without a large standalone MLOps team, a managed
  endpoint around a self-hosted model was the right point on that spectrum.
- **Instance sizing and auto-scaling for real daily usage, not spiky demo traffic.** Two clients'
  regulatory-affairs teams generating and reviewing sections throughout the working day is a
  materially different load profile from a demo or a batch job: sustained request volume during
  business hours, near-zero overnight, and periodic bursts around protocol amendment deadlines. That
  argues for a Sagemaker endpoint config with **auto-scaling policies tied to invocation count /
  latency** (scale out GPU instances as concurrent regulatory-writer sessions increase, scale back down
  overnight rather than paying for peak capacity around the clock) rather than a single fixed-size
  instance sized for worst-case load. Given LLaMA 3's GPU memory footprint (Quantization section
  above), the instance family choice itself (e.g., GPU-backed instance types with enough VRAM for the
  quantized model plus KV cache headroom at the expected concurrency) is a sizing decision made against
  real, measured request patterns, not guessed once and left alone.
- **Data residency/privacy for two competing clients was the actual reason to self-host, not a
  secondary concern.** Eli Lilly and AstraZeneca compete directly in several therapeutic areas, and
  their unpublished clinical trial protocols are exactly the kind of content neither company would
  accept being sent to a shared third-party inference API, let alone a provider whose infrastructure
  or logging practices are outside the platform's control. Hosting the fine-tuned model on a Sagemaker
  endpoint **inside AWS infrastructure the platform's own IAM roles and VPC controls govern** — with
  per-client isolation extended down to the inference layer itself (dedicated or namespace-isolated
  endpoints, see the per-client isolation discussion in `00-README.md`) — is what made it possible to
  credibly tell both clients their protocol content never left a controlled environment, let alone
  reached a third party. That guarantee is much harder to make convincingly about a closed third-party
  API, however strong that provider's own security posture might be, simply because the data has to
  leave your control boundary at all. This is the concrete, production version of the "data residency"
  argument made in the abstract earlier in this chapter — it's not a hypothetical regulatory concern,
  it's the specific reason self-hosting was non-negotiable for this client base.

## Tying it back

When asked "why LLaMA 3 and not just GPT-4 via Azure OpenAI," the strongest answer isn't "open-source
is better" in the abstract — it's the specific chain: clinical Protocol content is sensitive and
sponsor-confidential, so keeping inference and fine-tuning inside a controlled environment is a real
requirement, not a preference; the platform needs deep, per-module fine-tuning control (Chapter 02)
that a closed API doesn't expose as directly; and the sustained inference volume across many trials
and modules makes self-hosted infrastructure cost-competitive over time. That's a reasoned tradeoff
analysis, not a technology preference — and it's exactly what a senior interviewer is listening for.
