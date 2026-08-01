# 08 — Claude's Architecture vs. GPT-4's Architecture

## Read this chapter's framing before its content

Every other chapter in this course hedges specific numbers and outcomes while stating certain
architectural and product facts plainly (Claude is closed-weight, Azure AI Foundry is multi-vendor, and
so on). This chapter has the same discipline, applied to a topic where the temptation to overclaim is
much stronger — because "what's the architectural difference between Claude and GPT-4" is a genuinely
common interview question, and a wrong-but-confident answer is worse than an honest "I don't know that
part, and here's why no one outside Anthropic or OpenAI does either."

The single most important, most defensible point in this entire chapter is the one below — read it
first, and treat everything after it as commentary on top of that foundation, not a workaround for it.

## What's actually known vs. what's undisclosed

**Both Claude and GPT-4 (and its successors, GPT-4o and later) are proprietary, closed-weight,
decoder-only Transformer-based large language models.** That much is safe to state as settled fact —
both are text-in, text-out autoregressive Transformer models trained at large scale, served only via
hosted API.

Past that, **neither Anthropic nor OpenAI has publicly disclosed exact parameter counts, layer counts,
attention-head configurations, tokenizer internals beyond what's needed to use the API, or other core
architectural specifics for these models.** No official whitepaper lists GPT-4's parameter count. No
official whitepaper lists Claude's parameter count. Anyone confidently stating a specific number for
either is repeating a leak, a rumor, or a guess dressed up as fact — and that applies as much to
internal engineering conversation as it does to a public interview answer.

The contrast that makes this point land is **genuinely open-weight models**, where the comparison above
is simply not available. Meta's LLaMA family, Mistral/Mixtral, and DeepSeek all publish architecture
details — parameter counts, layer configurations, attention mechanisms, training data composition at a
meaningful level of detail — in technical papers and model cards anyone can read. Course 11 covers a
real deployment of one of these, LLaMA 3, in depth (see
[`03-open-source-llms-llama3-deployment.md`](../11-Indegene-GenAI-Regulatory-Platform/03-open-source-llms-llama3-deployment.md))
— and courses 13 and 14 in this curriculum cover two more genuinely open, published architectures,
worth a direct look if the contrast needs a second and third example.

The axis that actually separates these two categories of model is not "how capable is it" or "how
expensive is it." It's **closed-weight-and-undisclosed vs. open-weight-and-published** — and Claude and
GPT-4 sit unambiguously on the same, undisclosed side of that line. This is the same
self-hostable-vs-API-only distinction Chapter 1 draws for a different purpose; here it's specifically
about what's known about internals, not about who can run the weights.

## Alignment philosophy: Constitutional AI vs. RLHF

Anthropic has publicly described **Constitutional AI (CAI)** as a documented methodological choice: an
alignment approach that trains the model against a written set of principles (a "constitution"),
combined with AI-generated feedback in a reinforcement-learning loop — RLAIF (Reinforcement Learning
from AI Feedback) — rather than relying as heavily on large volumes of human-labeled preference data at
every stage. This is publicly described as philosophically distinct from OpenAI's original, primarily
human-feedback-based RLHF (Reinforcement Learning from Human Feedback) approach to aligning GPT-4-class
models, where human raters directly label or rank model outputs to train a reward model.

The honest caveat, worth stating in the same breath: **OpenAI has also evolved its own safety and
alignment techniques over time**, and the RLHF-vs-CAI framing that was clean at each method's initial
publication is less clean as a live, current-state comparison. Both companies iterate on alignment
methodology continuously, and neither publishes enough detail for an outsider to draw a precise,
up-to-date technical boundary between what each currently does under the hood.

The defensible claim is narrower than "Claude and GPT-4 are aligned completely differently": it's
"Anthropic has publicly committed to and named a distinct alignment philosophy (principle-based,
AI-feedback-assisted), which is a real, documented difference in *stated approach*, not a precise,
verified claim about current internal training pipelines at either company."

## Context window and long-document handling

Anthropic has publicly and consistently emphasized **long context windows** as a product differentiator
across successive Claude generations, and has published product materials and research specifically
about long-context behavior — this is a real, safe pattern to name. It's directly relevant to this
course's own research hypothesis (see `00-README.md` and Chapter 3): long-context handling was the
original reason Claude was worth evaluating for the case-narrative synthesis task in the first place,
before this course's later pivot toward the multi-agent underwriting system below.

What's **not** safe to do in an interview is commit to a specific current context-window token count as
a memorized fact. Context-window sizes get revised across model releases on both sides faster than any
static interview-prep document can track, and citing last year's number as current is a small but
noticeable credibility hit. The correct, durable framing: **"Claude has historically led on published
context-window size as a stated product differentiator — check the current published specs before
quoting an exact number in an interview, because that number moves."**

## Extended thinking vs. OpenAI's separate reasoning-model line

Anthropic has publicly announced an **"extended thinking"** capability in later Claude models — an
inference-time mode where the model allocates additional tokens to internal reasoning before producing
its final answer, trading latency and cost for improved performance on harder problems. This is
genuinely announced and real; the honest hedge is on version-specific mechanics (exactly how it's
configured, which models support which variant of it) rather than on whether it exists at all.

OpenAI has, separately, maintained a distinct **"reasoning model" product line** — historically launched
under names like "o1" and extended since — that is architecturally and product-line separate from the
mainline GPT-4/GPT-4o-class chat models, using extended chain-of-thought computation at inference time
as a defining feature of that specific product line rather than a mode within the mainline chat models.
That is a real, documented product-line distinction worth naming directly, with the standard hedge
attached: **verify the current product-line structure before quoting it, since this is exactly the kind
of thing that shifts with each vendor's release cadence.**

The contrast worth drawing explicitly, because it's the most interesting architectural-shaped
observation this chapter can honestly make: Anthropic's public framing treats extended thinking as **a
mode within one model family** — something you turn on for a given request against the same underlying
Claude model. OpenAI's public framing has, at least historically, treated deep reasoning as **a separate
product line** with its own model identifiers, distinct from the mainline chat models. That's a real
difference in how each company has chosen to expose reasoning-depth control to developers — it is not a
claim about which underlying training or architectural approach is "better," which is not something
either company's public disclosures actually let you evaluate.

## Computer use and why it matters to this course's pivot

Anthropic has publicly announced **"computer use"** — an agentic capability letting Claude interact with
a computer via screenshots and coordinate-based mouse/keyboard actions, enabling it to operate software
the way a human would rather than only through purpose-built API integrations. This is genuinely
announced, not rumored, and it's the most directly relevant fact in this whole chapter to the rest of
this course: it's real, public evidence that Anthropic has invested specifically in agentic, tool-using,
multi-step task execution as a first-class capability area, not just chat quality.

Combined with the publicly-announced extended thinking capability above and Claude's
tool-use/function-calling support (already covered in this course's Chapter 1, not re-explained here),
this is the concrete, defensible basis for choosing Claude as the backbone of the multi-agent Credit
Underwriting Assistant this course's pivot describes, starting in `00-README.md`: a system built from
five cooperating agents that plan, call tools, and hand off work to one another is exactly the kind of
task shape where publicly-announced strengths in careful multi-step reasoning and agentic tool-use are
most directly relevant — as distinct from a single-turn chatbot (Course 1) or a single-shot document
generator (Course 2), where GPT-4/Azure OpenAI was the uncontroversial enterprise default and no
particular agentic capability was being tested at all.

Both companies also support tool use/function calling more generally (Course 1's Chapter 5 covers this
pattern in depth) and ship vision-capable model variants, and both offer a tiered lineup of models — a
fast/cheap tier for high-volume work and a frontier/expensive tier for the hardest tasks, described here
by shape rather than by current name or price, both of which move too fast for a static document to
track reliably.

## How to answer "what's the architectural difference" without overclaiming

This is itself worth treating as an interview skill, not just a caveat to append. A strong answer to
"what's the architectural difference between Claude and GPT-4" does three things, in order:

1. **States what's actually known, confidently.** Both are closed-weight, decoder-only Transformer
   models; neither vendor discloses parameter counts or core architectural specifics. That's a real,
   confident, correct answer to the literal question — and naming genuinely open-weight alternatives
   (LLaMA, Mistral, DeepSeek) shows you know what "actually disclosed" looks like by contrast.
2. **Names the real, publicly-documented differences in approach and product surface** — alignment
   philosophy (CAI/RLAIF vs. RLHF, both evolving), context-window emphasis as a stated product
   differentiator, extended thinking as a mode vs. a separate reasoning-model product line, and
   computer-use as an agentic capability — without pretending any of these reveal internal model
   architecture. These are documented product and methodology facts, not leaked architecture details.
3. **Explicitly declines to fill the gap with a rumor.** If MoE (mixture-of-experts) comes up for GPT-4
   — and it's a common enough interview tangent that it's worth being ready for — the honest answer is:
   *widely reported and speculated via unofficial leaks, never officially confirmed by OpenAI.* Don't
   repeat it as settled fact, and don't speculate about Claude's architecture in the analogous way
   either; the correct answer for both is "undisclosed."

## Tying it back

The reason this chapter exists in this course, and not as a standalone trivia sheet, is that it's the
connective tissue between two things this course now argues together: Claude's genuinely undisclosed
internals mean "we chose Claude for its architecture" is not a claim this course can honestly make — but
Claude's *publicly announced capabilities* (extended thinking, computer use, tool use) are real,
checkable, and directly relevant to a task shape — multi-agent orchestration — where those specific,
documented strengths matter more than they do for a single-turn chatbot or a single-shot generator.

That's the sharpened version of this course's original premise, from "let's try a different vendor" to
"let's evaluate Claude specifically where its known strengths are most relevant" — and it's the frame
the rest of this course, starting with the pivot in `00-README.md`, builds on.
