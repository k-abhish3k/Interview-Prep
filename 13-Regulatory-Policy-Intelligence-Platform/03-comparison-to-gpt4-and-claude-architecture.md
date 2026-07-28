# 03 — How Mixtral's Architecture Compares to GPT-4's and Claude's

## Why this chapter exists

This is the explicit "how does this differ from GPT-4/Claude" comparison an interviewer who's just
heard Chapter 1's confident architectural walkthrough will naturally ask next. It's also the chapter
where this course connects most directly to course 3, which covers the Claude-vs-GPT-4 architecture
comparison from the other side (course 3, chapter 08) — that chapter goes deep on Claude and GPT-4
specifically; this one leads with the meta-point Chapter 1 already established and then compares on the
axes this course can actually speak to with confidence.

## The meta-point, stated first because it's the strongest part of the answer

**Mixtral's architecture is fully published and independently verifiable. GPT-4's and Claude's are
not.** That's not a rhetorical flourish — it's a literal, checkable difference in what's publicly known:

- Everything in Chapter 1 — decoder-only Transformer, Grouped-Query Attention, Sliding Window
  Attention, 8 experts per layer, top-2 routing, ~47B total / ~13B active parameters — is stated in
  Mistral's own published technical report and model card. Anyone can read it, verify it, and reproduce
  the described behavior against the publicly released weights.
- OpenAI has not published GPT-4's architecture. Details like parameter count, whether it uses a
  Mixture-of-Experts design, and its training methodology are not officially disclosed.
- Anthropic has not published Claude's architecture. As course 3 establishes plainly, Claude is
  closed-weight and API-only — there is no version of "download Claude's weights," and there is
  likewise no published architectural specification comparable to what Mistral has released for
  Mixtral.

For a project whose central rationale (Chapter 2, Reason 3) is model-risk transparency and audit, this
isn't a side note — it's the architectural fact that makes the whole transparency argument possible in
the first place. You cannot hash-verify, independently evaluate, or architecturally audit a model whose
structure was never disclosed to begin with; you can only audit its external behavior. Mixtral's
publication status is a precondition for everything Chapter 2's third reason claims.

## Comparing on the axes this course can actually speak to with confidence

| Axis | Mixtral 8x7B | GPT-4 | Claude |
|---|---|---|---|
| Architecture disclosure | Fully published (Mistral's technical report, model card) | Undisclosed by OpenAI | Undisclosed by Anthropic |
| Mixture-of-Experts | Confirmed sparse MoE, 8 experts/layer, top-2 routing (Chapter 1) | Widely rumored/reported to use MoE, but **never officially confirmed by OpenAI** — state this precisely, not as fact | Fully undisclosed; no public claim either way |
| Weight access | Open weights, Apache 2.0 license — downloadable, self-hostable | Closed weights, API-only | Closed weights, API-only |
| Fine-tuning surface | Full weight access — LoRA/QLoRA or full fine-tuning on infrastructure you control (Chapter 5) | Vendor-controlled fine-tuning API surface (where offered), on OpenAI's terms and infrastructure | Vendor-controlled fine-tuning surface (where offered), on Anthropic's terms and infrastructure |
| Deployment options | Self-hosted (Azure ML/AKS, this course) **or** managed API via Azure AI Foundry — your choice | API only — Azure OpenAI Service or OpenAI's own API | API only — direct from Anthropic or via a cloud marketplace including Azure AI Foundry |
| License | Apache 2.0 — permissive, enables commercial self-hosting/on-prem/private-cloud deployment without per-seat or per-token vendor lock-in | Proprietary — access governed entirely by OpenAI's terms of service | Proprietary — access governed entirely by Anthropic's terms of service |

A few of these rows deserve their own sentence rather than just sitting in the table:

**The GPT-4 MoE row is worth getting exactly right, because overclaiming it is an easy, common mistake.**
There has been substantial public reporting and speculation that GPT-4 uses a Mixture-of-Experts
architecture — but OpenAI has never officially confirmed this, and stating it as settled fact in an
interview is the kind of overclaim a sharp interviewer will catch and discount your credibility for.
The precise, defensible statement is: "GPT-4 is widely reported and speculated to use MoE, but that's
never been officially confirmed by OpenAI — unlike Mixtral, where it's a documented, verifiable fact."
That precision is itself the point of this chapter.

**The license row is a real, practical difference, not just a legal footnote.** Apache 2.0 is a
permissive open-source license — it allows commercial use, modification, self-hosting, and
redistribution with minimal restriction. That's what makes it legally straightforward for a bank's
Capco engagement to download Mixtral's weights and deploy them inside their own Azure tenant at all;
there is no equivalent path for GPT-4 or Claude, whose access terms are entirely vendor-controlled API
agreements, not a weights license.

## The honest capability caveat — say this plainly, don't oversell

A strong interview answer here does not end with "and that's why Mixtral is the better model." It
isn't, necessarily, on raw capability. Mixtral 8x7B is very likely **not** at true frontier capability
parity with the latest GPT-4-class or Claude-class models on every benchmark — those closed, frontier
models are trained at far larger scale, with far more resources devoted to alignment, safety tuning,
and capability optimization, and they generally lead open-weight models of Mixtral's generation on
demanding reasoning, coding, and instruction-following benchmarks.

The honest, defensible argument for choosing Mixtral on this project was never "it's smarter." It was
**architectural transparency, cost at volume, and fine-tuning control** (Chapter 2) — three concrete,
workload-specific engineering tradeoffs that favor a self-hostable, fully-documented, fine-tunable open
model over a stronger but closed, API-only, black-box one, for this specific document-intelligence
workload at this specific volume and governance requirement. A regulatory-classification task, domain-
fine-tuned on a specific taxonomy, is also a task where a smaller, well-adapted model can close much of
the capability gap to a larger general-purpose model anyway — narrow, well-scoped tasks are exactly
where fine-tuning control (Reason 2) pays for itself, versus a task that genuinely needs frontier
general reasoning across a huge span of unpredictable inputs, where a closed frontier model's raw
capability edge would matter far more. Saying this plainly — "here's why we didn't need the smartest
model available, and why that's a defensible engineering call, not a compromise we're hiding" — is a
stronger answer than overselling Mixtral's raw capability and getting caught out by a follow-up
benchmark question.

## Tying It Back

If asked "how does this compare to GPT-4 or Claude, architecturally": lead with the fact that Mixtral's
architecture is fully published and Mistral's own claims are independently verifiable, while GPT-4's and
Claude's are not (course 3, chapter 08, covers the Claude/GPT-4 side of that comparison in depth — point
to it rather than re-deriving it); compare precisely on MoE (confirmed for Mixtral, rumored-but-
unconfirmed for GPT-4, undisclosed for Claude), weight access (open vs. closed, both cases), and license
(Apache 2.0's permissiveness vs. no equivalent for either closed model); and close by being honest that
this is not a capability argument — it's a transparency, cost, and control argument, made for a
regulated client with real model-risk-governance needs a black-box API, however capable, cannot satisfy.
