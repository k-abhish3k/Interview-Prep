# 03 — Comparison to GPT-4 and Claude Architecture

## Lead with the meta-point

Course 3's chapter 08 establishes, from the GPT-4/Claude side, that both of those models are
proprietary, closed-weight, decoder-only Transformer models whose core architectural specifics —
parameter counts, layer configurations, attention-mechanism internals — are simply not disclosed by
either OpenAI or Anthropic. That chapter names DeepSeek (alongside LLaMA and Mistral/Mixtral) as one
of the genuinely open-weight counter-examples where that comparison is actually available, and points
to this course as the place to look for the DeepSeek side of it in depth. This chapter is that: the
explicit, side-by-side "how does this differ from GPT-4/Claude" comparison, built from Chapter 1's
account of DeepSeek's published architecture.

The single most important framing point, worth stating first every time this question comes up in an
interview: **the axis that actually separates these models isn't capability, it's disclosure.**
DeepSeek's architecture — Multi-head Latent Attention, fine-grained Mixture-of-Experts, its
auxiliary-loss-free load-balancing strategy — is published in technical reports anyone can read and
independently verify. GPT-4's and Claude's architectures are not disclosed at all beyond "proprietary,
closed-weight, decoder-only Transformer." That's not a subtle distinction dressed up as a big one —
it's the entire reason Chapter 1 can state its claims with total confidence while course 3's chapter
08 has to spend most of its length explaining what it honestly doesn't know.

## Attention mechanism: MLA (documented) vs. undisclosed (both)

DeepSeek's approach to the KV-cache memory problem — Multi-head Latent Attention, compressing
per-head keys and values into a shared lower-dimensional latent representation before caching,
reconstructing full-size keys/values from that compressed representation at attention time — is a
specific, named, published technique (Chapter 1). Neither GPT-4 nor Claude has disclosed what
attention mechanism variant either model uses internally. That doesn't mean neither company has ever
worked on KV-cache efficiency — inference cost is a first-order concern for any provider running
models at frontier scale, and it would be unsurprising if both have internal techniques addressing
exactly this problem — but **nothing has been officially published confirming what those techniques
are** for either model. The honest, defensible comparison is: DeepSeek's specific KV-cache-compression
approach is a documented, distinctive, nameable technique; GPT-4's and Claude's attention mechanisms
are, in the technical-disclosure sense, simply unknown quantities from the outside.

## Mixture-of-Experts: confirmed and detailed (DeepSeek) vs. rumored (GPT-4) vs. fully undisclosed (Claude)

This is the axis worth being most precise about, because it's the one where sloppy answers most
often slide into repeating a rumor as settled fact — exactly the trap course 3's chapter 08 warns
against explicitly. DeepSeek-V3's fine-grained Mixture-of-Experts design, including its specific
expert-granularity choice (many smaller experts plus always-on shared experts) and its
auxiliary-loss-free load-balancing mechanism, is confirmed, named, and described at real technical
depth in DeepSeek's own published papers (Chapter 1). That's about as strong a disclosure posture as
an architecture claim can have.

GPT-4 using an MoE architecture is, per course 3's chapter 08, **widely reported and speculated via
unofficial leaks, never officially confirmed by OpenAI**. It's a common enough interview tangent to
be worth being ready for, and the correct move is exactly what that chapter recommends: name it as
persistent, credible-sounding rumor, and stop there — don't repeat it as fact, and don't fill in
invented specifics (expert count, activation pattern, routing strategy) that have never been
confirmed by anyone in a position to know. Claude's architecture, per the same chapter, has no
comparable rumor mill attached to it in wide public circulation at all — it sits even further into
"undisclosed" than GPT-4's MoE speculation, which at least has *some* leaked signal behind it, however
unofficial.

Ranking these three by disclosure confidence, from most to least: DeepSeek's MoE design (officially
published, technically detailed) > GPT-4's MoE status (unofficially rumored, never confirmed) >
Claude's architecture generally (essentially no public technical speculation with any credibility
behind it). That ordering is itself a useful thing to be able to state plainly in an interview — it
shows you're tracking *disclosure confidence* as its own dimension, not just repeating "they're all
big transformer models" as if that were the whole answer.

## Open-weight and self-hostable (DeepSeek) vs. API-only and closed-weight (both GPT-4 and Claude)

This is the most consequential practical difference for this project specifically, and it's a
disclosure question with real operational teeth, not just a trivia point: DeepSeek's weights are
downloadable and deployable under a permissive open license (Chapter 1), which is the precondition
for everything this course's architecture depends on — self-hosting on SageMaker (Chapter 4),
LoRA fine-tuning onto therapeutic-area vocabulary (Chapter 5), and a fixed, checksummed, versioned
model artifact an audit trail can point to (Chapter 6, Chapter 7). GPT-4 and Claude are both
**API-only, closed-weight** — you can call them, you cannot download, inspect, fine-tune at the
weight level, or self-host either one, full stop, regardless of how much you're willing to pay or how
large your enterprise agreement is. That's not a disclosure nuance the way the attention-mechanism and
MoE comparisons above are — it's a hard, binary capability difference, and it's the concrete reason
this project's central architectural choice (Chapter 2) was only possible with a model like DeepSeek
in the first place.

## Being honest about capability, not just architecture

It would overstate this chapter's case to let "DeepSeek is more transparent" slide into "DeepSeek is
therefore better." It isn't guaranteed to be, and saying so plainly is part of a strong answer here,
matching the same honest framing course 13's equivalent chapter uses for Mixtral in its own Azure/
banking context: DeepSeek, like other open-weight models, is not guaranteed to match the latest
GPT-4-class or Claude-class frontier model on every benchmark, and open-weight model releases in
general have historically traded some frontier capability for the openness and cost/control
advantages that make them worth choosing for a specific workload. The argument for DeepSeek in this
project was never "it's the best model available, full stop" — it's the three-part case Chapter 2
builds: architectural transparency and auditability (this chapter, and Chapter 6), cost economics at
this workload's extreme volume (Chapter 2, Reason 1), and fine-tuning control for a specific domain
task (Chapter 2, Reason 2; Chapter 5). Those are real, defensible, workload-specific reasons — they
are not a claim of unqualified superiority over GPT-4 or Claude on general capability, and conflating
"right choice for this workload" with "best model in the world" is exactly the overclaim this chapter
is built to avoid.

## The compressed interview answer

> "DeepSeek publishes its architecture — Multi-head Latent Attention for compressing the KV cache,
> and a fine-grained Mixture-of-Experts design with a specific, documented load-balancing mechanism —
> in technical reports anyone can verify. GPT-4 and Claude are both closed-weight, and neither
> OpenAI nor Anthropic has disclosed comparable architectural detail — GPT-4's MoE status is
> widely rumored but never officially confirmed, and Claude's architecture has essentially no
> credible public speculation attached to it at all. The practical consequence that mattered for this
> project is that DeepSeek's weights are open and self-hostable, and GPT-4's and Claude's are not —
> you cannot download, fine-tune, or self-host either of them at any price. That's why DeepSeek was
> the model this project could self-host and fine-tune onto therapeutic-area vocabulary in the first
> place. None of this is a claim that DeepSeek beats GPT-4 or Claude on capability — it very well
> might not, on a lot of benchmarks. The case for it here is architectural transparency, cost
> economics at extreme screening volume, and fine-tuning control for a specific domain task, not
> unqualified superiority."

## Tying It Back

The strongest version of this comparison holds two things true at once without letting either one
collapse into the other: DeepSeek's architecture is confidently, verifiably known, and GPT-4's and
Claude's are not — that's a real, stateable fact, not a hedge. And separately, "known architecture"
is not the same claim as "better model" — DeepSeek's case for this project rests on transparency,
cost, and fine-tuning control for a specific workload, not on a capability claim this chapter has no
basis to make. Cross-reference course 3's chapter 08 for the full GPT-4/Claude-side treatment of what
is and isn't known about those two models — this chapter doesn't re-derive that content, it builds on
it from the DeepSeek side.
