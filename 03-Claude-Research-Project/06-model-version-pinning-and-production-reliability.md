# 06 — Model Version Pinning and Production Reliability

## The pattern this chapter is an instance of

Every course in this curriculum with a versioning/staleness chapter is answering some version of the
same question: *how do you know what you're looking at right now is still the thing you think it is,
and not something that quietly changed underneath you?* Course 1 asks it about RAG-retrieved knowledge.
Course 2 asks it about a case narrative's underlying data. Course 4 asks it about a monitoring baseline.
This course asks it about something none of the others do: **the model itself** — and, because this
system is a five-agent pipeline rather than one model call, it asks it at five different points at
once, any one of which can drift independently of the others. A production claim like "the Underwriting
Memo Drafting Agent's output quality has been stable for the last quarter" is only as trustworthy as the
guarantee that "the model behind that agent" meant the same specific, fixed model across every request
that produced that claim — and that guarantee is not automatic. It is a well-known, real problem in
applied production ML generally, not something specific to this course's setup.

## (a) The naive assumption

The naive version of this deployment, run without thinking about versioning at all, looks like this:
point each agent's deployment configuration at a Claude endpoint referenced by its model family name (or
a deployment/endpoint name that resolves to "whatever Anthropic currently serves under that name"), run
the system in production for some number of weeks or months, and report that "the multi-agent system has
been running on Claude with stable memo-quality scores." That sentence sounds complete. It is missing
the one piece of information that makes it verifiable, auditable, or even meaningfully re-checkable six
months later: **which specific model version each agent was actually calling, on any given day.**

## (b) Why this is a real production-reliability gap, not a pedantic detail

Vendors — Anthropic included — ship new model versions behind existing deployment names and endpoint
identifiers on an ongoing basis, sometimes as a deliberate, disclosed upgrade and sometimes as a routine
model refresh with less fanfare. If any agent's deployment configuration references a model by an
unpinned, "always latest" identifier rather than an explicit, dated version string, then a model update
rolled out by the vendor **silently changes what that agent is actually doing** — without a single line
of the orchestration code changing, without an error, without anything in the pipeline surfacing a
signal that anything happened at all. And because this is a five-agent system, that silent change can
happen to one agent and not the others, at different times, which is a strictly harder version of the
single-model staleness problem this pattern started from.

Concretely, imagine a production quarter where the Risk Assessment Agent's memo-quality score, tracked
by the monitoring pattern course 4 established, drifts downward partway through. If the underlying model
version behind that one agent changed mid-quarter — while the other four agents' deployment
configurations stayed on their original pinned versions — a monitoring dashboard that only tracks
"average memo quality across the pipeline" has no way to attribute that drift to the right cause. Worse,
if the version change happened to coincide with, say, a shift in the mix of industries appearing in the
loan-application pipeline that quarter, a real engineer could easily misattribute a score shift to the
wrong cause entirely — blaming applicant-mix drift for a change that was actually a model-version
change, or vice versa, because nothing in the data distinguishes the two explanations without an
explicit version field to check. That's not a hypothetical footnote; it's exactly the kind of silent
confound that turns a well-monitored production system into an unreliable one, and it happens without
anyone doing anything wrong in the orchestration code — the code behaved exactly as configured, it was
the configuration itself that left a reliability gap open, independently, per agent.

## (c) Two different questions that get conflated

The specific conflation worth naming explicitly, because it's the root of the problem: **"the API is
still reachable and returning valid responses" is a completely different question from "it's still the
same model I validated this agent against before."** An unpinned endpoint answers the first question
with a reassuring "yes" indefinitely — requests keep succeeding, responses keep coming back well-formed,
nothing about the integration looks broken, and the Supervisor keeps happily routing work to an agent
whose underlying model quietly changed weeks ago. It says nothing at all about the second question. A
monitoring setup that only checks "is each agent's endpoint up" (a reasonable, necessary check on its
own, and one Chapter 7 covers) provides zero signal about whether the model behind that agent is still
the one its evaluation results and production quality baseline were actually established against.
Treating "the integration works" as proof that "the agent's behavior is still what we validated" is the
exact mistake this chapter is naming — the same category of mistake as assuming a document is current
because the link to it still resolves, which is precisely the analog course 2's own staleness chapter
draws for a different kind of data going stale underneath a system that has no way of noticing.

## (d) The fix: explicit pinning per agent, plus a fleet-wide deprecation check

Three concrete, implementable pieces close this gap, and all three matter — pinning alone without
logging, or logging without a deprecation check, each leaves a real hole, and in a multi-agent system all
three need to run **per agent**, not once for the pipeline as a whole:

1. **Pin an explicit, dated model version identifier in every agent's deployment configuration** — not
   an "always latest" alias, and not a single shared config value that's assumed to apply uniformly to
   all five agents (they can legitimately be pinned to different tiers or even different versions,
   Chapter 1 notes, and the pinning discipline has to account for that rather than assume uniformity).
   The Azure AI Foundry deployment each agent references (or the direct Anthropic model identifier, if
   accessed that way) should point at a specific, versioned model string, so a request sent to that agent
   today and a request sent to it in six months are guaranteed to hit the same model, unless someone
   deliberately changes that agent's configuration.
2. **Log that same version identifier alongside every single agent invocation** — tagged with which
   agent made the call, not just the version string. This is the piece that makes a later audit possible:
   if the Risk Assessment Agent's quality score looks like it shifted partway through a quarter, the
   per-agent version field is what lets someone check whether a model change happened to *that specific
   agent* at that point, rather than guessing across all five.
3. **Run a scheduled re-validation check, per agent**, that queries the vendor's model catalog or
   deprecation announcements for each pinned version and flags — proactively, before it becomes an
   incident — when any agent's pinned version is approaching a vendor-announced deprecation date. A
   pinned model version isn't a "set once, forget forever" configuration for any of the five agents;
   vendors retire older model versions on their own timeline, and a pinned agent deployment that's never
   revisited eventually points at a version that's about to stop being served at all, discovered only
   when that one agent's requests start failing — the worst possible way to find out, and, in this
   system, potentially the worst possible *place* to find out, since a failing agent has to be caught by
   the failure-isolation design Chapter 7 covers rather than being allowed to take the whole pipeline
   down. `notebooks/04_model_version_pinning_demo.ipynb` implements exactly this pattern: a deployment
   config that pins an explicit version, an evaluation-result logger that stamps every row with that
   version, and a scheduled check against a mocked vendor deprecation calendar that raises a warning
   inside a configurable window before the deprecation date — the same pattern this chapter argues needs
   to run once per agent in the real fleet, not once for the system as a whole.

## (e) Tying forward to course 2's staleness chapter

Course 2's chapter 5 asks "what happens when new transactions arrive or a KYC profile changes while a
narrative is drafted or sitting in review" — the same underlying question this chapter asks, applied to
a model instead of a document, and now applied five times over instead of once. The mechanism differs
(course 2 needs a data-freshness check tied to a customer record's last-modified timestamp; this chapter
needs a model-version field logged on every agent invocation plus a per-agent deprecation-monitoring
check), but the shape of the problem, and the shape of the fix, is identical: **make the thing that can
silently go stale carry an explicit, checkable version marker, and build a process that actively watches
that marker rather than assuming it never changes.** If an interviewer draws the parallel themselves —
"isn't this the same pattern as your other project's staleness problem, just applied to a model, and now
to five models instead of one" — the correct answer is an enthusiastic yes, because recognizing that a
production-reliability problem and a data-freshness problem are structurally the same problem, wearing
different clothes, is exactly the kind of pattern-recognition a senior engineer is expected to
demonstrate.

## Tying it back

The one-sentence version to have ready: **"the multi-agent system runs on Claude" is an incomplete,
unverifiable claim; "each of the five agents runs on an explicitly pinned Claude version, logged on
every invocation with the agent that made it, and a scheduled per-agent check flags any of them before
Anthropic's announced deprecation date" is a production system someone else could actually audit,
reproduce, or trust six months later.** That difference — between an integration that merely keeps
working and a production system whose behavior is still actually what it was validated to be — is the
whole point of this chapter, and it's a cheap, mechanical fix relative to how much reliability it buys a
system with this many independently-moving parts.
