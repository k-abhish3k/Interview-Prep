# 08 — Development Challenge: Cross-Functional Collaboration

## Why this challenge is different from the other seven

Chapters 01–07 are all, at core, technical problems with technical fixes — scaling, isolation,
monitoring, cost, data, evaluation, integration. This chapter is different: it's about working with
stakeholders who have **genuine veto power over what ships**, whose objections aren't a technical
disagreement to be debugged but a different, legitimate set of priorities that has to be
reconciled, not overridden. For the banking engagements, that's a **model-risk/compliance
function** applying SR 11-7-style governance (course 4). For the pharma engagements, that's
**medical/legal/regulatory (MLR) affairs reviewers** who own final sign-off on any generated
content before it reaches an external audience. An interviewer asking "tell me about a time you
disagreed with a stakeholder" or "how do you handle pushback from a non-engineering team" is
listening for exactly this chapter.

## What these stakeholders actually optimize for, and why it's not "no"

It's easy to frame compliance/MLR friction as an obstacle to engineering velocity, and that framing
tends to produce a weak interview answer (it reads as "the compliance team slowed me down" rather
than "I understood their constraint and worked within it"). The more accurate — and more useful —
framing:

- **A bank's model-risk function is legally and reputationally accountable** for what a deployed
  model does. Their job is to make sure every model in production has documented validation,
  monitoring, and a clear owner — not to block features for their own sake. A monitoring gap or an
  undocumented prompt change is a real problem from their seat, even if it looks like paperwork from
  an engineering seat.
- **A pharma MLR reviewer is accountable for regulatory and legal exposure** on every piece of
  content that reaches a healthcare provider or patient — a generated claim that isn't properly
  substantiated, or a document that skips the required review step, is a genuine regulatory risk,
  not a process formality.
- Both groups are trying to prevent a class of failure that's rare but expensive when it happens —
  which structurally puts them in tension with an engineering team optimizing for shipping speed,
  even when both sides want the same ultimate outcome: a good, safe product in production.

## Where the friction actually shows up

- **Review-cycle timing**: an engineering team wants to ship a prompt/feature change on a sprint
  cadence; a compliance/MLR review has its own cadence (weekly review board, documentation
  turnaround time) that doesn't match, creating a structural bottleneck if not planned around.
- **Evidence burden**: what counts as "proof" that a change is safe differs — an engineer might
  consider a passing eval-set run sufficient, while a reviewer needs a documented methodology, a
  human-reviewable sample, and a clear owner for ongoing monitoring (this is exactly why chapter 06's
  golden eval set is built with the reviewer's evidence needs in mind from the start).
- **Risk appetite framing**: engineers naturally think in terms of "is this good enough," reviewers
  think in terms of "what's the worst case if this is wrong, and how would we know" — a genuinely
  different mental model that a good working relationship has to bridge, not just tolerate.

## What actually works: making governance a design input, not a gate at the end

The strongest version of this collaboration treats the compliance/MLR function as **a stakeholder
with requirements to design for, not a final gate to survive** — analogous to designing for a
non-functional requirement like latency or security from the start rather than trying to bolt it
on before launch:

- Loop the model-risk or MLR reviewer in during design, not just before launch — showing them the
  planned evaluation methodology (chapter 06) early means their requirements shape the golden set
  from the start, instead of surfacing as a rejection after the set is already built.
- Build the audit trail and human-in-the-loop review points (chapter 02) into the architecture as a
  first-class feature, not an afterthought bolted on to satisfy a late-stage review comment.
- Give the reviewing function their own visibility (a dashboard, a documented methodology, direct
  access to a sample of flagged cases) rather than making every question require a meeting — this is
  the same instinct as the client-visible usage dashboard in chapter 04, applied to governance
  instead of cost.

## Illustrative friction point: a disagreement over launch readiness, resolved productively

> **Illustrative, plausible scenario — not a verified account of a specific engagement.** Built
> from the real governance dynamics of SR 11-7-style banking model risk and pharma MLR review.

**Situation.** Ahead of launching a prompt-engineering update to a banking AI Assistant intended to
improve response quality on a class of previously-weak queries (course 1/2 context), the model-risk
reviewer assigned to the assistant declined to sign off on the original proposed timeline, on the
grounds that the change hadn't been evaluated against the full harmfulness/robustness metric suite
from course 4 — only against the accuracy/completeness metrics the engineering team had prioritized,
because the harmfulness suite took meaningfully longer to run and the team was targeting a sprint-end
release date.

**Task.** I needed to resolve a genuine disagreement about launch readiness between engineering
velocity and governance rigor, without either quietly skipping a required check to hit the date or
treating the reviewer's objection as an obstacle to route around.

**Action.** Rather than arguing the date, I asked the reviewer to walk through specifically what
risk the harmfulness/robustness suite was meant to catch that the accuracy/completeness metrics
wouldn't, which clarified that their concern was specifically about the new prompt's behavior on
adversarial/edge-case inputs — a legitimate gap the faster-running metrics genuinely didn't cover.
I proposed a compromise that addressed the actual risk rather than the calendar: ship the change
behind a limited rollout (a small percentage of traffic, via the same deployment-slot mechanism used
for rollback in chapter 03) while the full harmfulness/robustness suite ran in parallel against the
limited-rollout traffic, with a pre-agreed threshold for either promoting to full rollout or rolling
back. This gave the reviewer real evidence before full exposure, rather than either a delayed launch
or an ungated one.

**Result.** *(Illustrative)* The full evaluation suite completed within a few days and passed
against the limited-rollout traffic, and the change was promoted to full rollout roughly a week
after the original target date rather than requiring an indefinite delay. The limited-rollout
pattern — ship small, evaluate in parallel, then promote — became a standing option offered whenever
an engineering deadline and a governance review timeline were in tension, rather than defaulting to
"wait for the full review" or "ship anyway" as the only two choices.

## Talking points this chapter sets up

- Reframing compliance/MLR pushback as a legitimate, accountable stakeholder's requirement rather
  than a process obstacle — this framing alone is often what separates a strong answer from a weak
  one on this question.
- Involving governance stakeholders during design rather than only before launch, so their
  requirements shape the artifact (the eval set, the audit trail) rather than gating it after the
  fact.
- A concrete de-escalation pattern (limited rollout + parallel evaluation + pre-agreed promotion
  threshold) that resolves a velocity-versus-rigor disagreement without either side simply losing.
- Treating "what risk does this check actually catch" as the right question to ask a reviewer,
  rather than treating the requirement as arbitrary.
