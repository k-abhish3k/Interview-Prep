# 02 — Why Self-Hosted Open-Weight vs. a Managed API (Including Azure AI Foundry's Own Mixtral)

## Why this chapter exists

Every other course in this curriculum's Capco track (1, 2, 4, 5, 6) uses Azure OpenAI, a managed API,
without much debate — for a chatbot or an AML copilot at that traffic volume, a managed API is the
obviously correct choice, and courses 1 and 3 say so. This course is the deliberate exception, and the
first, most important interview question it has to answer well is: **why not just do what every other
Capco project here did?** The honest answer is three separate, distinct arguments — cost at volume,
fine-tuning control, and weight-level transparency — and blurring them into one vague "self-hosting is
better" claim is exactly the mistake to avoid. Each argument has its own reasoning, its own limits, and
its own honest caveats.

## A precise version of the alternative being rejected

Before the three reasons, it's worth being exact about what's being *turned down*, because it's a
stronger, more specific option than "just use a closed model." **Mistral's models, including Mixtral,
are themselves available through Azure AI Foundry's model catalog** — the same multi-vendor catalog
course 3 covers for Claude — as a managed, hosted API. That means the naive "just use the API" option
here isn't "give up the open-weight model and use GPT-4 instead" — it's "get the *exact same model
weights*, Mixtral, served to you as an API call, with Microsoft operating the infrastructure." That's a
genuinely appealing middle path: same underlying model, none of the hosting burden. This project
evaluated it and chose self-hosting anyway, for reasons that don't go away just because the API vendor
happens to be serving the same weights — which is the whole point of separating the three reasons
below from a generic "open beats closed" argument.

## Reason 1: Cost at volume

**The claim.** Per-token API pricing — whether Azure OpenAI or Azure AI Foundry's managed Mixtral
offering — scales close to linearly with tokens processed. Self-hosted inference has a cost structure
dominated by **fixed compute** (GPU-hours, whether the endpoint is busy or idle, plus the infrastructure
around it) rather than a per-token meter. Those are two different cost curves, and at low volume the
linear (API) curve wins — you pay only for what you use, with zero idle cost. At high, sustained
volume, the fixed-cost (self-hosted) curve wins, because its cost stops growing with volume once
GPU capacity is provisioned, while the API's cost keeps climbing.

**The shape of the tradeoff, not a verified cost model.** It's worth stating explicitly, the way a
strong interview answer would: the specific breakeven volume depends on real GPU pricing, real API
pricing, real utilization of the provisioned GPU capacity, and real token volume — none of which should
be presented as a precise, memorized number without having actually run the calculation against current
prices. What's defensible to say with confidence is the *shape*: sketch two lines on a chart, one
roughly linear in token volume (the API), one roughly flat until you need to add another GPU instance,
then stepping up (self-hosted) — the crossover point is somewhere in the middle, and a platform running
continuous, high-volume document processing (tens of thousands of pages a week, per document module)
is exactly the kind of workload profile that sits well past that crossover, in the region where
self-hosting wins. A platform running occasional, bursty, low-volume calls is exactly the kind of
workload that should stay on a managed API regardless of any of the other two reasons below — cost
alone can settle the question in the other direction at low volume.

**The honest caveat.** Fixed-compute cost isn't free just because it isn't metered per token — GPU
instances cost money whether or not they're fully utilized, so a self-hosted deployment that's
badly underutilized (provisioned for peak load, mostly idle) can lose the cost argument entirely.
Chapter 4's autoscaling discussion exists specifically to keep utilization, and therefore this cost
argument, honest in practice rather than just in theory.

## Reason 2: Fine-tuning control

**The claim.** A managed API — even Azure AI Foundry's own Mixtral offering — exposes fine-tuning, if
it exposes it at all, as a constrained, vendor-controlled service: a fixed set of hyperparameters, a
fixed training data format, no visibility into exactly how the adaptation is applied under the hood,
and no ability to inspect or export the resulting adapted weights as an independent artifact. Owning
the base model's weights directly means full access to **LoRA/QLoRA** (or, if genuinely needed, full-
parameter) fine-tuning, on infrastructure the client controls, with complete visibility into the
training process, the training data, and the resulting adapter or checkpoint.

For this platform specifically, the client's requirement was concrete, not abstract: a model fine-tuned
on their **own, proprietary regulatory taxonomy and internal policy classification scheme** — a
category structure that is itself confidential, evolves over time, and is not something a generic
API vendor's fine-tuning service is designed around. Course 11's LoRA/QLoRA chapter
(`02-instruction-tuning-and-llm-finetuning.md`) covers the mechanics of low-rank adaptation in depth —
the same technique applies directly here, and Chapter 5 of this course walks through applying it to
this platform's taxonomy-classification task specifically rather than re-deriving LoRA from first
principles again.

**The honest caveat.** Fine-tuning control is only valuable if the organization actually intends to use
it. If a project's real need turns out to be "occasionally adjust behavior via a system prompt," full
weight access is a solution to a problem that doesn't exist yet, and the fixed infrastructure cost
(Reason 1) is being paid for a capability going unused. This project's need was real and specific
(taxonomy fine-tuning, expected to recur as the taxonomy evolves — see Chapter 7), which is what makes
the reason defensible here rather than a generic justification reached for by default.

## Reason 3: Model transparency and audit

**The claim.** This is the reason that ties most directly to course 4's model-risk-monitoring themes,
and it's the one most specific to a *regulated banking client's model-risk function*, not a generic
engineering preference. A model-risk function that has to formally characterize a model's behavior —
what it does well, where it fails, how it responds to adversarial or edge-case input, whether its
behavior has changed since the last review — benefits enormously from being able to run its own
controlled evaluations against a **fixed, known, weight-level artifact** that cannot silently change
underneath them. A closed-weight API, by contrast, is a black box that could be updated by the vendor
without notice (course 3's chapter 06 covers exactly this risk for Claude — an unpinned API endpoint
silently serving a new model version mid-evaluation), and even a *version-pinned* API still gives the
model-risk function no way to inspect the actual weights, only the model's external input/output
behavior.

Self-hosting Mixtral means the client's model-risk function can hold an exact, hash-verified copy of
the weights (Chapter 6 covers how, concretely — checksumming against Mistral's published model card and
provenance tracking in a model registry), run its own evaluation suite against that exact artifact on
its own infrastructure, and know with certainty that the artifact being evaluated is the same one
running in production, forward-referenced explicitly by the model-version-pinning discipline this
course's own Chapter 7 builds on top of.

**The honest caveat.** Weight-level access does not automatically mean a model-risk function can fully
*explain* the model's behavior — Mixtral is still a large neural network, and having the weights
doesn't make its internals mechanistically interpretable in the way, say, a Shapash/LIME explanation
does for a simpler classical model (course 4, chapter 3). What weight-level access does buy is
**reproducibility and independent evaluability**: the ability to run the same fixed artifact repeatedly
under controlled conditions, rather than depending on a vendor's word that nothing has changed. That's
a real, meaningful form of transparency — but it's worth being precise that it's not the same claim as
"we can fully explain every output the model produces."

## The three reasons together, stated once, precisely

If asked to compress this chapter into one breath: **cost at volume** argues for self-hosting on pure
economics once volume clears a real breakeven point; **fine-tuning control** argues for it because the
client's requirement (proprietary taxonomy fine-tuning) needs direct weight access a managed API's
fine-tuning surface doesn't offer; **transparency/audit** argues for it because the client's model-risk
function specifically wanted a fixed, hash-verifiable, independently-evaluable artifact rather than a
black-box API call, even a version-pinned one. Any one of these alone might not justify the operational
burden of self-hosting (Chapter 4, Chapter 8); together, for this specific client and this specific
workload, they did.

## Tying It Back

The strongest version of this answer in an interview names Azure AI Foundry's managed Mixtral offering
explicitly and explains why it wasn't chosen — that's the detail that shows you understand this was a
genuine architectural evaluation, not a reflexive "self-hosted is always better" position. It also shows
you can hold three separate arguments in your head at once without collapsing them into one, which is
exactly the kind of precision a senior technical interviewer is listening for.
