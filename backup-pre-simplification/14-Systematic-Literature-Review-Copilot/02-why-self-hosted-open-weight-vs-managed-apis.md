# 02 — Why Self-Hosted Open-Weight vs. Managed APIs

## Why this chapter matters

"Why not just use Bedrock, or Azure OpenAI, or any managed API?" is the first architecture question
an interviewer will ask about this project, and it deserves better than a one-line answer. This
chapter builds the three-part rationale introduced in `00-README.md` out to the depth needed to
survive follow-up questions — cost at extreme volume, fine-tuning control, and regulatory
auditability — treating each as a genuinely separate argument rather than one blurred justification.
Keeping them distinct matters because an interviewer who pushes on one ("okay, but couldn't Bedrock's
fine-tuning APIs handle that?") is testing whether you actually understand which argument that
pushback is attacking, or whether you're going to retreat to a different one without noticing you've
changed the subject.

## Reason 1: Cost economics at extreme volume

A systematic literature review's screening workload has a shape that's genuinely unusual relative to
most GenAI applications this curriculum covers. Course 1's chatbot and course 8's virtual liaison
platform are interactive, conversational, human-paced — a person types a question, waits a few
seconds, reads an answer. SLR screening is the opposite: a **single review** can involve screening
**several thousand to tens of thousands of abstracts**, each one a discrete, independent, non-
interactive model call applying the same inclusion/exclusion criteria. A large pharma client running
Indegene's SLR copilot across multiple therapeutic areas, with reviews refreshed periodically as new
literature publishes, accumulates that volume many times over across a year.

The rough shape of the breakeven argument, without inventing precise numbers (which would depend on
actual negotiated API pricing and actual GPU instance costs at the time of a real deployment,
neither of which is stable enough to hardcode into an interview-prep document): **per-token managed
API pricing is a marginal cost that scales linearly with call volume**, with no floor and no
ceiling — screen twice as many abstracts, pay roughly twice as much, forever. **Self-hosted inference
on a SageMaker endpoint has a cost structure dominated by GPU instance-hours** — a largely fixed cost
for the capacity you provision, which you pay whether you're running it at 10% or 100% utilization,
but whose *per-abstract* cost drops as utilization rises, because you're spreading a mostly-fixed
hourly cost across more work. Those two cost curves cross somewhere: below some sustained volume, the
managed API is cheaper because you're not paying for idle GPU capacity; above it, self-hosting is
cheaper because the fixed infrastructure cost is being amortized across enough throughput that its
per-unit cost undercuts linear per-token pricing. The concrete, defensible claim for this project
isn't "self-hosting is always cheaper" — it's "**a platform running sustained, high-volume batch
screening across many concurrent reviews for many therapeutic areas, essentially continuously, is
firmly on the side of that breakeven curve where self-hosted infrastructure wins**," which is a
volume-and-utilization argument, not a blanket claim about self-hosting versus APIs in general.
Chapter 4's autoscaling-for-bursty-batch-workloads discussion is the operational engineering that
makes sure that utilization assumption actually holds in practice rather than being an idealized
average.

## Reason 2: Fine-tuning for domain reasoning

Screening decisions aren't a generic reading-comprehension task — they depend on the model correctly
applying a specific client's **therapeutic-area vocabulary** (the way an oncology review's
inclusion criteria talk about tumor staging and response criteria is a different vocabulary than a
cardiology review's talk about ejection fraction and MACE endpoints) and the particular shape of
**that review's inclusion/exclusion criteria**, which are themselves refined as a review proceeds
(Chapter 7). A general-purpose instruction-tuned model prompted with criteria text alone can get a
reasonable fraction of screening decisions right, but "reasonable fraction" is not the bar for a
process feeding a regulatory submission — the gap between "usually right" and "reliably right,
measured against a gold standard" (Chapter 5's screening precision/recall evaluation) is exactly what
weight-level fine-tuning is positioned to close.

Course 11's chapter 02 covers the LoRA/QLoRA mechanism in full — the frozen base weights, the low-rank
adapter matrices, the parameter-efficiency argument — and this project applies that same mechanism to
a different target: not clinical-document generation format/register (course 11's ICF/PLPS/SOC use
case), but **screening-decision reasoning conditioned on therapeutic-area vocabulary and a specific
review's criteria patterns** (Chapter 5 here). The reason this needs weight-level access rather than
prompt engineering alone is the same reason course 11 gives: getting a model to reliably apply a
nuanced, domain-specific judgment across thousands of repeated calls is a different, harder bar than
getting it to usually apply that judgment when prompted well. A closed API constrains fine-tuning to
whatever surface the provider chooses to expose, on their terms and their infrastructure, if they
expose one at all for the model version in question — a self-hosted open-weight model gives direct,
unconstrained access to the weights that adaptation actually needs to touch.

## Reason 3: Auditability for regulatory submissions

This is the argument most specific to this project's stakes, and the one worth leading with when the
question is framed around regulatory risk rather than engineering cost. An SLR that feeds a
regulatory submission is not a convenience tool — it's part of the evidentiary basis for a claim a
regulator might scrutinize, which means the process producing it needs to be able to answer, for any
given screening decision, a precise version of "what, exactly, made this call, and can we reproduce
why." That's a **fixed, versioned, fully characterizable model artifact** question: which exact model
checkpoint, which exact configuration (temperature, prompt template, any fine-tuned adapter in use),
and which exact criteria version (Chapter 7) produced a given screening decision on a given abstract,
at a given point in time.

A third-party managed API is a genuinely worse fit for that requirement, independent of how good the
provider's own security or quality practices are — the core issue is that **a managed API endpoint
can be silently updated by its provider**. A model identifier that looks stable from the caller's
side can have its underlying weights, safety tuning, or serving configuration changed by the vendor
without the caller's initiation or necessarily even without change-log visibility into exactly what
shifted, at exactly what implication for behavior. For an interactive chatbot, that's often an
acceptable, even desirable, tradeoff — you generally want your provider's steady quality
improvements. For a process whose defensibility depends on being able to say precisely what produced
a specific evidentiary judgment, that same "could change under you at any time" property is a
liability, not a convenience. A self-hosted model artifact — a specific set of downloaded, checksummed
weights (Chapter 6's provenance-verification discussion), deployed at a specific configuration,
pinned and never silently altered — is the kind of thing an organization can point to and say
precisely what it was, indefinitely, regardless of what any vendor ships next. Chapter 7's model-
version-tracking design is the concrete mechanism that makes this guarantee operationally real rather
than aspirational.

## The three reasons, kept distinct

A quick gut-check worth having ready for a follow-up that tries to collapse these into one: if a
managed API vendor offered fine-tuning access tomorrow, would that eliminate the case for
self-hosting? No — Reason 3 (auditability) would remain, because the vendor could still silently
update the underlying serving stack around a fine-tuned adapter, and Reason 1 (cost at this volume)
would likely still favor self-hosting past the breakeven point regardless of whether fine-tuning was
available on the API side. Conversely, if cost were a non-issue (Reason 1 moot), Reasons 2 and 3
would still independently justify self-hosting for a regulatory-submission-supporting process. That
independence is exactly what makes this a strong three-part answer rather than one argument
restated three ways — each reason survives even if the other two were somehow neutralized.

## Tying It Back

"Why self-host instead of using a managed API" has a stronger answer than "control" or "it's
cheaper" in the abstract: it's cost economics that specifically favor self-hosting at this
workload's sustained, extreme-volume shape (Reason 1); fine-tuning access that specifically matters
because screening reliability depends on domain-adapted reasoning, not generic prompting (Reason 2,
extending course 11's LoRA content to a new use case); and auditability that specifically matters
because this process feeds regulatory submissions, where a fixed, versioned, characterizable model
artifact is a real requirement a silently-updatable third-party API structurally can't satisfy
(Reason 3, tying forward to Chapter 6's provenance work and Chapter 7's staleness tracking). Naming
all three, and being ready to defend each independently, is what separates this from a generic
"open-source is better" talking point.
