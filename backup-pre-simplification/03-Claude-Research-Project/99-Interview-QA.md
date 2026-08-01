# 99 — Interview Q&A: Multi-Agent Credit Underwriting Assistant (Claude Research Project)

Read this last, after the chapters are fresh. This course now describes a production, client-delivery
system — the same tier as courses 1 and 2 — built around a task shape neither of those courses covers:
multiple cooperating agents rather than one model in one call. Expect the open-source correction to come
up the moment Claude is mentioned by name, and expect "why multiple agents instead of one good prompt"
to be one of the first follow-ups once the architecture is on the table. Both are handled head-on, right
up front, the same way course 2 leads with "why isn't this fully automated" and course 5 leads with its
own factual correction.

---

### 1. I heard this was an open-source Claude model — is that right?

This is the single most important question in this whole course to answer smoothly and confidently,
because getting it wrong undermines credibility on everything that follows it.

> "No — and that's worth correcting directly, because it changes what the interesting part of this
> project actually is. Claude is proprietary and closed-weight: Anthropic doesn't release the model
> weights, so there's no version of downloading Claude and running it on our own hardware, the way you
> genuinely can with an open-weight model like Meta's LLaMA — which, incidentally, is a model I've also
> worked with directly, on a pharma platform that self-hosted a fine-tuned LLaMA 3 on a Sagemaker
> endpoint. That contrast is actually useful: LLaMA is open-weight and self-hostable; Claude is
> proprietary and API-only, reachable either directly from Anthropic or through a cloud marketplace. In
> this project's case, every one of our five agents reaches it through Azure AI Foundry's model
> catalog — inside the same Azure tenant as our existing enterprise Azure OpenAI deployment, behind the
> same Private Endpoints and Azure AD gate, not through a separate, lighter-weight environment. So the
> accurate framing is: a production multi-agent underwriting system where every agent calls Claude, a
> proprietary, API-only model accessed via Azure AI Foundry, chosen specifically because its
> publicly-announced strengths in careful reasoning and agentic tool use are relevant to a multi-agent
> task shape — not an open-source model evaluation at all."

**Follow-up to be ready for:** *"What's the actual difference between Azure OpenAI Service and Azure AI
Foundry, then?"* → Answer 6 below has the full version; the one-sentence version is: Azure OpenAI
Service is a single-vendor product that can only ever serve OpenAI's models — that's not a
configuration limit, it's what the product is — while Azure AI Foundry's Model Catalog is a
multi-vendor marketplace that includes OpenAI alongside Anthropic, Meta, Mistral, and others, and it's
the only correct way to reach Claude from inside an Azure environment.

---

### 2. (Behavioral) Tell me about this project.

> "This is a production system built for the same banking client context as our other Capco banking
> deliverables — a Multi-Agent Credit Underwriting Assistant that accelerates commercial loan
> underwriting. The task doesn't fit a single-model pattern: it needs a borrower's financials spread
> into ratios, checked against a written credit policy, combined with external risk signals, and
> synthesized into a defensible, cited recommendation — four genuinely distinct kinds of analysis. I
> designed it as five cooperating agents: a Financial Spreading Agent and a Risk Assessment Agent that
> run in parallel because neither depends on the other's output; a Credit Policy Compliance Agent and an
> Underwriting Memo Drafting Agent that run sequentially because each needs an earlier agent's output;
> and a Supervisor/Orchestrator Agent, built as a LangGraph state graph, that routes the work, decides
> when a case has enough information to proceed, and escalates ambiguous or incomplete cases to a human
> loan officer instead of guessing.
>
> I chose Claude as the backbone specifically because the task shape — multi-step, tool-mediated,
> multi-agent coordination — plays to capabilities Anthropic has actually announced publicly: extended
> reasoning and agentic, tool-driven task execution. That's a sharper argument than 'let's try a
> different vendor' — it's 'let's evaluate this vendor specifically where its known strengths are most
> relevant,' which is the same reasoning course 1 and course 2 didn't need, because a single-turn
> chatbot and a single-shot document generator aren't testing agentic capability at all.
>
> I deployed the orchestrator on Azure — Container Apps, alongside our existing App Service
> deployments — inside the same VNet, Azure AD, and Azure Monitor setup as every other production system
> in our banking practice, with each agent calling Claude through Azure AI Foundry. I rolled it out in
> shadow mode first, running the full pipeline against real loan applications without ever showing its
> output to a real loan officer, then in a disclosed canary once the evaluation was signed off — and at
> every stage, the human loan officer's sign-off requirement stayed absolute: the system produces a
> draft recommendation, never an autonomous decision."
>
> *(Illustrative result, replace with real numbers)* "The system materially reduced average
> time-to-draft-recommendation, with loan officers approving most drafted memos with only minor edits,
> while every recommendation still passed through mandatory human sign-off."

---

### 3. How do you keep the rest of your production banking systems separate and safe while rolling out a brand-new five-agent system in the same environment?

One of the two or three most important questions this course needs to survive — featured prominently
for exactly that reason.

> "The core design principle is: same governance boundary, but strict, structural isolation from
> anything that can affect a real decision, until a formal sign-off says otherwise. Concretely, the
> Supervisor and all four specialist agents sit inside the same VNet, behind the same Private Endpoints,
> gated by the same Azure AD tenant, and traced into the same Azure Monitor instance as our production
> Azure OpenAI deployments — I didn't stand up a separate, lighter-weight environment just because it
> was a new agentic pattern. But each agent authenticates under its own scoped RBAC role, so no single
> agent's identity has a path to bypass the Supervisor and reach a real loan officer's queue directly.
>
> The rollout itself is staged. First, shadow mode: the full five-agent pipeline runs against real loan
> application intake in parallel with the existing manual underwriting process, its output is logged and
> scored, and it's never shown to a real loan officer and never influences a real decision — it's a
> read-only observer riding alongside the real process. I made sure that isolation was true by
> construction, not by convention: if any specialist agent fails — timeout, error, malformed output — the
> Supervisor degrades that section to an explicitly-flagged gap rather than letting the failure
> propagate, and if the orchestration graph itself hits a hard step-count ceiling, the run terminates and
> escalates to a human rather than looping. I have fault-injection tests that verify both of those
> directly.
>
> Only after that shadow-mode evaluation is formally reviewed and signed off does a second stage begin —
> a small canary of real applications actually drafted by the multi-agent system — and even then, the
> review workflow itself doesn't change at all. The loan officer still has to approve every
> recommendation before it becomes a real decision; the only difference is they're explicitly told the
> draft was AI-produced and by which agents. Nothing about introducing a five-agent system ever shortcuts
> the human sign-off requirement — it just adds more disclosed information to what the reviewer sees."

**Follow-up to be ready for:** *"Doesn't running five agents instead of one model call multiply your LLM
cost?"* → Yes, directly — a five-agent pipeline is, in the straightforward case, five calls per
application where a single-shot system needs one, and I was upfront about that as a real,
budget-relevant number. Two things keep it from being worse than it has to be: parallelizing the two
agents with no data dependency (recovers latency, not cost), and letting the Supervisor escalate early,
before running the full pipeline, on cases that were always going to need a human anyway (recovers real
cost, not just latency) — Chapter 7 covers both in depth.

---

### 4. How did you handle data compliance for a brand-new AI vendor, now powering five different agents?

The other of the two or three most important questions this course needs to survive.

> "The fact I made sure not to gloss over: even though every agent reaches Claude from inside our
> existing, already-approved Azure environment — same network boundary, same governance — Anthropic
> itself is a genuinely new third-party vendor with no prior relationship with our organization, and
> that's true for every one of the five agents' calls, not just one. 'Same Azure environment' is true
> and important for the network and governance layer, but it says nothing about vendor risk, and
> treating those as the same thing would have been a real mistake.
>
> So I treated the vendor risk assessment for Anthropic as a real, necessary gate — not paperwork to
> route around because the rollout needed to move fast. And because that assessment realistically takes
> longer than standing up the Azure AI Foundry resource itself, I designed the work so it didn't need to
> wait on it: the earliest development and evaluation phase ran entirely on synthetic loan-application
> data — realistic financial-ratio distributions, synthetic policy queries, synthetic risk narratives —
> without containing any real borrower's information, and later on de-identified historical application
> data. Only after the vendor assessment concluded and a data processing agreement was actually in place
> did the system touch real, identifiable applicant data, across any agent.
>
> I also made a point of not assuming our existing no-training-on-customer-data guarantee with Microsoft
> automatically carried over to Anthropic just because every agent reaches Claude through the same cloud
> marketplace — it doesn't, automatically, and I confirmed the specific commercial terms in writing. And
> I pushed for more scrutiny on data residency for this new vendor, not less, even though the Azure
> resource sits in the same region as everything else — because a new vendor means new, unanswered
> questions about where their own backend and subprocessors sit, for every agent's traffic, which our
> existing, already-vetted Microsoft relationship had already had answered years ago."

**Follow-up to be ready for:** *"Wasn't running on synthetic data just a workaround because you couldn't
get real applicant data cleared in time?"* → No, and it's worth pushing back on that framing directly —
it was the correct order of operations for introducing any new AI vendor, powering a five-agent
pipeline, to a regulated environment. It meant the riskiest early phase of the work — debugging the
integration across five separate agents, tuning the orchestration and evaluation design itself —
happened on data where a mistake had zero real consequence.

---

### 5. How do you know the model behind each agent today is the same one you validated when the system first went live?

Featured prominently, and the natural technical follow-up to Q4 — this is the model-version-pinning
production-reliability question, now with a multi-agent twist.

> "That's exactly the gap that made me build explicit, per-agent version pinning into this from early
> on. Vendors, Anthropic included, ship new model versions behind existing deployment names on an
> ongoing basis, and if any one of the five agents' deployment configs references a model by an
> unpinned, 'always latest' identifier, a vendor-side update can silently change what that specific
> agent is doing — while the other four stay exactly as they were. That's a strictly harder version of
> the single-model staleness problem, because the drift can happen to one agent at a time, and a
> pipeline-wide quality metric can hide which agent actually changed.
>
> There's a distinction worth being precise about here too: 'each agent's endpoint is still reachable
> and returning valid responses' is a completely different question from 'it's still the same model I
> validated that agent against before.' An unpinned endpoint answers the first question with a
> reassuring yes forever and tells you nothing about the second. So every agent's deployment config pins
> an explicit, dated model version identifier — not an alias that resolves to whatever's current — every
> single agent invocation gets logged with both its own agent identity and that pinned version string,
> and a scheduled check runs per agent against the vendor's deprecation announcements, flagging any of
> the five proactively, before it stops being served, rather than discovering it when one agent's
> requests start failing.
>
> It's the same underlying pattern as a staleness problem I dealt with on a related project — knowing
> whether a case narrative's underlying customer data is still current, or whether a document a RAG
> system retrieved is still the latest version — just applied to five models instead of one document.
> The fix is structurally identical: give the thing that can silently go stale an explicit, checkable
> version marker, per agent, and build a process that actively watches every one of those markers instead
> of assuming none of them ever change."

**Follow-up to be ready for:** *"What would you actually do if the deprecation check fired for just one
of the five agents?"* → Treat that agent's version change as isolated: segment its evaluation history by
version, decide whether pre- and post-change data for that agent can be reported together, and re-pin it
deliberately — without touching the other four agents' configs, since there's no reason a version change
in one agent should force a re-validation of agents that didn't change at all.

---

### 6. What's the actual technical difference between Azure OpenAI Service and Azure AI Foundry?

> "Azure OpenAI Service is a dedicated, single-vendor product — it exists specifically to serve OpenAI's
> models inside Azure's compliance boundary, and there's no setting or SKU that gets it to serve a
> non-OpenAI model, because that's not what the product is for. Azure AI Foundry — it used to be called
> Azure AI Studio — is Azure's general-purpose, multi-vendor model catalog and MLOps platform: it lists
> OpenAI's models alongside Anthropic's Claude, Meta's Llama family, Mistral, and others, each with its
> own deployment options. So 'the same Azure environment, but reaching Claude' can only mean Azure AI
> Foundry — it's not a hidden Azure OpenAI feature, and it's not a reason to go outside Azure entirely.
> That distinction is also why our architecture has two separate resources sitting side by side in the
> same VNet — the existing Azure OpenAI deployment, unchanged, and a new Azure AI Foundry endpoint every
> agent calls for Claude — rather than one resource somehow serving both; they're genuinely different
> Azure products with different deployment lifecycles, even though they share the same network and
> governance boundary."

---

### 7. Walk me through a concrete difference between calling Azure OpenAI and calling Claude through Azure AI Foundry — at the API/code level, not just conceptually.

> "The clearest one is the response shape. Azure OpenAI's Chat Completions API returns
> `response.choices[0].message.content` — a plain string, one fixed field. Claude's Messages API returns
> `response.content` as a list of typed content blocks — text blocks, tool-use blocks, and others — so
> you filter for the block type you actually want rather than reading one guaranteed string field. That's
> a structural difference, not a naming difference: a Claude response can legitimately carry more than one
> block in a single turn, and code that assumes 'the response is always exactly one string' breaks the
> first time that shape shows up — which matters more here than it would for a single model call, since
> that parsing logic is shared across five different agents' handlers.
>
> System-prompt handling is another concrete one — Chat Completions treats the system instruction as just
> another entry in the `messages` array with `role: system`; the Messages API pulls it out into its own
> top-level `system` parameter, entirely separate from the `messages` list. I actually had a real bug from
> exactly this difference — an early version of our shared adapter layer had been written and tested
> against one agent's Claude branch first, and the other agents' branches hadn't been separately verified
> before the first end-to-end evaluation run, so a mapping error concatenated system instructions into the
> first user message instead of routing them into the dedicated `system` parameter, across every agent
> that reused the shared code. Chapter 7 of the course has the full writeup — a golden-request test
> asserting the exact payload shape sent for each agent would have caught it before it ever produced a
> misleading evaluation score."

---

### 8. Why would multiple agents actually help this specific task, and where's the nuance you'd want to be careful about overclaiming on the context-window side?

> "The mechanical story: the Underwriting Memo Drafting Agent has to synthesize three upstream agents'
> full outputs — financial ratios, policy citations, risk rationale — into one cited memo. For a large
> commercial borrower with several years of financials and a policy corpus with a lot of applicable
> covenant sections, that combined shared state can be big enough that it doesn't fit without compressing
> one or more agents' output before the handoff. That compression step has a real, specific cost:
> information that looks unremarkable in one agent's output, summarized in isolation, can matter a lot
> once read alongside a related detail from a different agent — and by the time the memo is drafted, that
> connection is gone, because no step in the pipeline ever held both agents' full outputs at once. A
> model with a genuinely large context window that can hold all three upstream outputs in full removes
> that specific failure mode.
>
> But I was careful not to let that turn into 'bigger context window equals better cross-agent
> synthesis, full stop.' Having room to fit everything is a different capability from actually attending
> well to every part of it once it's there — models can show real degradation on content buried in the
> middle of a long combined prompt, even with a huge nominal window, which is sometimes called the 'lost
> in the middle' effect. In this system's shape, that risk is specifically that whichever upstream
> agent's output happens to land in the middle of the assembled context gets under-weighted relative to
> whichever landed first or last — for reasons that have nothing to do with which agent's findings
> actually matter most. So having room for everything doesn't guarantee the Memo Drafting Agent used all
> of it well, which is exactly why the evaluation methodology in Chapter 2 has a dedicated
> cross-agent-consistency metric that checks whether every upstream agent's findings actually made it
> into the final memo, rather than assuming they did because the token math worked out."

---

### 9. Walk me through how you designed the evaluation to avoid bias — both from the automated scoring and from the human reviewers — and how that's different from evaluating a single model call.

> "The biggest change from a single-model evaluation is what's actually being measured. It's not just
> 'is the final memo well-written' — it's whether the orchestration itself behaved correctly: did the
> Supervisor route and dispatch agents in the right order, did each agent's tool calls come back
> well-formed and successful, and did the final memo actually incorporate every upstream agent's
> findings rather than silently dropping one. I built those as explicit, mechanically-checkable metrics —
> routing correctness against a hand-labeled expected dispatch sequence, tool-call validity against a
> schema, and cross-agent consistency using the same claim-decomposition-and-check pattern our monitoring
> team's Ragas-style scorers use for faithfulness, just applied across agent boundaries instead of within
> one context window.
>
> For the parts that do need subjective judgment — is the final memo readable, does the Recommendation
> section actually follow from the evidence — I ran a genuinely blind side-by-side review, the same
> design as before: every memo a loan-officer reviewer saw was labeled 'System A' or 'System B,' never
> the actual system name, mapping randomized per case and held outside the reviewer's view until scoring
> was complete, presentation order randomized too, scored against a fixed rubric tied to the memo's own
> structure rather than an open-ended impression. And on the automated side, I made a point of not
> letting the judge model be Claude itself, since every agent in the pipeline runs on it — using the same
> model to judge its own agents' coordination would bias the judge toward whatever Claude's own notion of
> 'consistent' happens to be."

---

### 10. Why blind the human evaluation at all — isn't a loan officer's professional judgment trustworthy either way?

> "It's not a question of trustworthiness, it's a question of a very well-documented, universal cognitive
> bias — a reviewer who's told 'this memo was drafted by the new multi-agent system' will, even with the
> best intentions, tend to read it slightly more charitably or slightly more skeptically than they'd read
> the identical text with no label at all, purely because of the framing. That's not specific to loan
> officers; it's the same reason a clinical trial blinds which arm a patient is in. Blinding removes that
> channel entirely, so the score reflects the memo's actual quality, not the reviewer's prior expectation
> of which system should be better. It's worth being precise that this is a completely different design
> choice from what I did later, in the canary stage — there, I deliberately disclosed which system
> produced a draft, because a loan officer signing off on a real credit decision has a right to know an
> AI system, still early in its rollout, produced what they're looking at. Blinding is for getting an
> unbiased evaluation measurement; disclosure is for a real sign-off decision. Those are two different
> situations that call for opposite choices, and conflating them would be a mistake."

---

### 11. Walk me through, end to end, what happens when a loan application comes through the system in shadow mode.

> "A real loan application comes in and flows through the existing manual underwriting process exactly
> as it does today — nothing about that changes. In parallel, the same application's documents are also
> routed to the Supervisor, which dispatches the Financial Spreading Agent and Risk Assessment Agent
> concurrently, since neither depends on the other. Once the Financial Spreading Agent's ratios are
> available, the Compliance Agent is dispatched to check them against the credit policy corpus via Azure
> AI Search. Once all three specialist agents have returned — or explicitly failed, in which case their
> section is flagged as unavailable rather than silently dropped — the Underwriting Memo Drafting Agent
> synthesizes everything into a structured, cited draft memo. The whole run is bounded by a hard
> step-count and timeout ceiling enforced independently of the routing logic itself, so a bug in that
> logic can't cause an unbounded loop. Whatever comes out — a complete memo, a partial one with flagged
> gaps, or an orchestration-error escalation — gets logged with every agent's pinned model version
> attached, scored against the agent-task-completion metrics from Chapter 2, and written into the
> comparison dataset. At no point does any of this touch the real underwriter's independently-produced
> recommendation for the same application — that's the isolation guarantee I made sure held even under
> fault injection, not just under normal operation."

---

### 12. How would you scale this system up in production without blowing the LLM budget?

> "The lever I'd reach for first isn't 'add more agents' or 'run everything on the most capable tier' —
> it's making sure the Supervisor escalates early on cases that are always going to need a human, before
> paying for the full four-agent pipeline. A loan application missing required documentation doesn't
> need the Compliance Agent or the Memo Drafting Agent to run before the system can tell it's going to
> need human handling — recognizing that early is a real cost saving, not just a latency one. Beyond
> that: keep parallelizing every pair of agents without a data dependency rather than defaulting to
> sequential dispatch, reassess whether every agent genuinely needs the same model tier (a narrower,
> more mechanical step like structured financial extraction is a plausible candidate for a faster, cheaper
> tier if per-agent evaluation supports it), and watch the step-count ceiling's trigger rate as an
> operational signal — if it's firing more than expected, that's a sign the routing logic has a gap
> worth investigating before it becomes a bigger cost problem."

---

### 13. Tell me about a bug you found and fixed that was specific to comparing two LLM vendors, not a generic backend bug.

> "The token-counting one is a good, concrete example because it's genuinely specific to comparing two
> vendors, not something you'd hit calling just one API. Our cost-per-application comparison originally
> reused a tokenization utility that had been written for the Azure OpenAI integration and used OpenAI's
> own tokenizer under the hood. When every agent moved onto Claude, that same utility got reused across
> all five agents' calls without anyone separately verifying it was measuring the right thing — and it
> wasn't, because OpenAI's and Anthropic's tokenizers segment the same text differently. Applying
> OpenAI's tokenizer to Claude's output produced a token count that didn't match what Anthropic actually
> billed, and the resulting cost comparison systematically misrepresented the multi-agent system's real
> cost — in the run where this surfaced, understating it, compounded across five agents' worth of calls
> instead of one.
>
> What would have caught it sooner: cross-checking the locally-computed token count against the real
> usage figures each vendor's own API response returns — both APIs report actual usage — and alerting on
> any divergence beyond a small tolerance. That check would have flagged the mismatch on the very first
> agent's first Claude request, instead of after enough volume across the whole pipeline had built up for
> someone to notice the aggregate numbers looked implausibly favorable."

---

### 14. What's the most serious issue you ran into on this project, and how was it caught?

> "I'd rather answer this honestly than give the sanitized version, because I think how something like
> this gets caught and closed is the more useful thing to talk about than pretending it never happened.
> Before our vendor risk assessment for Anthropic had concluded, the multi-agent pipeline was already
> running in shadow mode, as designed, against synthetic data only. A logging change meant to enrich the
> comparison dataset with additional context metadata had a field-mapping error, and for a period of a
> few hours, a small number of shadow-mode requests from the Compliance Agent ended up including a
> retrieved-context field that — because of an edge case in how our synthetic-data generator had been
> seeded from a handful of real, de-identified loan applications as templates, and one template that
> hadn't been fully scrubbed of an identifying detail — carried a fragment of real applicant text into a
> request sent to Anthropic's infrastructure, ahead of the compliance clearance that's supposed to gate
> real data reaching that vendor at all.
>
> It was caught through a routine audit-log spot-check of shadow-mode requests, which in hindsight is a
> gap in itself — it should have been caught by a real-time PII/leakage scanner, not a spot-check, and
> that's a concrete change I made afterward. Once identified, I rolled back the logging change
> immediately, purged the affected requests and any cached responses, reported it through the same
> incident channel a real data breach would go through, and audited and hardened the synthetic-data
> generation pipeline so a seed template could no longer retain an identifying fragment in the first
> place. I'd call that a genuine near-miss, not a minor hiccup, and I think being direct about that,
> including what specifically closed the gap, is a better answer than pretending the safeguards worked
> perfectly the first time."

---

### 15. This all sounds expensive for a five-agent production system — how would you have handled it if the evaluation had come back "the multi-agent system isn't actually better than a single well-prompted model"?

> "That would have been a completely legitimate, valuable outcome, not a failure of the project. A
> negative or ambiguous result is exactly what a real evaluation is supposed to be able to produce — if I
> designed a comparison that could only ever come back positive, it wouldn't have been a real evaluation,
> it would have been a foregone conclusion dressed up as engineering rigor. The honest version of that
> outcome is: 'we tested whether decomposing this task into specialized agents actually improved
> coordination correctness and output quality enough to justify the added cost and latency, ran a
> rigorous evaluation against the agent-task-completion metrics and a blinded memo-quality review, and it
> didn't clear the bar' — and that's still a genuinely useful result, because it closes off a question
> the practice would otherwise have kept wondering about, with actual evidence rather than a hunch. I'd
> write that up with the same rigor as a positive result and not let the sunk cost of having built five
> agents pressure the conclusion in either direction."

---

### 16. The canary succeeded — what's the actual next step, does the system just quietly take over underwriting?

> "No, and I'd push back on that framing if it came up, because it's exactly the wrong instinct. A
> successful canary — a small percentage of real applications, disclosed as AI-drafted, still going
> through full human sign-off — justifies proposing a broader rollout, not an autonomous one. Any
> expansion of the percentage of real applications the system touches is its own explicit, reviewed
> decision, with its own monitoring checkpoint, not something that happens gradually and silently just
> because the canary went well. And at every stage, no matter how far the rollout expands, the system's
> output is a draft recommendation — the human loan officer's approval is what actually makes something a
> credit decision, and that doesn't change regardless of how much confidence the system has earned.
> Keeping that boundary explicit is part of what makes the whole rollout defensible to a risk-averse
> audience — nothing quietly graduates to a fully autonomous decision without an explicit, named decision
> point in between."

---

### 17. What would you change if you did this project again?

> "A few concrete things. First, I'd build the hard step-count and timeout ceiling on the orchestration
> graph from day one, not add it after a routing bug caused the Supervisor to loop on the Compliance
> Agent — that protection is a basic safety property of any supervisor-pattern system, not a defense
> against a bug nobody could have anticipated, and I shouldn't have needed an incident to build it.
> Second, I'd build the real-time PII/leakage scanner on the shadow-mode logging path from day one rather
> than relying on a periodic audit-log spot-check to catch a data-handling mistake. Third, I'd write the
> golden-request payload tests for every agent's Claude request shape before running a single real
> evaluation, not after a misleading first round of results forced me to go back and add them — 'does it
> run without throwing' is a much weaker test than 'does it send exactly the payload I intend,' and with
> five agents sharing adapter code, that gap affected all of them at once. Fourth, I'd design the
> per-agent model-version pinning and logging from the very first request rather than adding it partway
> through. And I'd push to get the vendor risk assessment for Anthropic started even earlier, in parallel
> with the earliest synthetic-data groundwork, since the assessment being the longer pole in the timeline
> was predictable from day one."

---

### 18. How is this project different from course 2's AML Alert Investigation Copilot — aren't they both just Claude-vs-GPT stories?

> "They're related in that both are production Capco banking systems and both involve evaluating a
> model backend choice, but the actual engineering problem is different, and I'd want to be precise about
> that if it came up. Course 2 is a single-shot structured-generation system: one retrieval pass across
> three source types, one model call that drafts a cited narrative, running on the enterprise Azure
> OpenAI deployment. This system is a genuinely different task shape — commercial loan underwriting
> doesn't decompose into one retrieval pass and one generation call, it needs several distinct kinds of
> analysis converging into one recommendation, which is why it's built as five cooperating agents instead
> of one model call, and why the backbone model choice (Claude, specifically for its publicly-announced
> agentic strengths) is a different, sharper argument than a generic vendor comparison. Course 2's
> structured-output and citation-grounding pattern is directly reused here, in the Underwriting Memo
> Drafting Agent — I didn't reinvent that — but the orchestration, the failure-isolation design, and the
> runaway-loop protection are all new problems this system has that course 2 never needed to solve,
> because course 2 never had more than one model call in flight at a time."

---

### 19. Why multiple agents instead of one really well-prompted model?

> "The honest answer: a single model call asked to spread financials, check policy, assess risk, and
> draft a cited memo all in one pass has to hold four different jobs in its attention at once, with one
> undifferentiated context mixing four kinds of source material. Specializing that into separate agents
> lets each one's system prompt and tool access be tuned purely for its own job — the Financial
> Spreading Agent's prompt is entirely about extraction accuracy, with no competing pressure from
> policy-threshold reasoning. It also buys real architectural properties a single call can't: agents can
> fail independently, so a Risk Assessment Agent outage doesn't have to take the whole pipeline down;
> independent agents can run in parallel where their outputs don't depend on each other, recovering real
> latency; and a Supervisor with an explicit completeness check can recognize 'I don't have enough here'
> and escalate to a human, rather than a single model silently doing its best across four jobs it was
> never given room to think through separately. I don't think that argument is unconditionally true for
> every task — for course 1's single-turn chatbot or course 2's single-shot generator, splitting into
> agents would have added coordination overhead for no real benefit. It's specifically true for a task
> that already decomposes into genuinely distinct sub-analyses, which commercial underwriting does and a
> single customer question doesn't."

---

### 20. How do you prevent runaway agent loops?

> "A supervisor pattern has a structural risk a single model call doesn't: nothing inherently stops it
> from routing back to an agent it already called if its own completion logic has a bug, or hits a case
> it wasn't designed for — there's no natural request boundary the way there is for one API call. The
> fix is a hard step-count limit and a wall-clock timeout enforced by the Supervisor's own control logic,
> independent of whatever the routing logic itself decides. If either limit is hit, the run terminates
> immediately, gets flagged distinctly from a normal completion or a normal escalation, and gets routed
> to a human with whatever partial state accumulated — it never keeps running on the theory that it'll
> probably terminate naturally soon. I actually have a real incident behind this answer, not just a
> theoretical design: an early routing bug misread a Compliance Agent response that was correctly empty
> — because that borrower's industry genuinely had no extra covenants to cite — as incomplete, and kept
> re-invoking the same agent, which kept correctly returning the same empty-but-valid result, for a long
> unbounded stretch before it was manually killed. If the step-count ceiling had existed from the start,
> that incident would have been a small, cheap, immediately-visible blip instead of an open-ended cost
> incident — which is exactly why that protection belongs in the system from day one, not added
> reactively after the first time it's needed."

---

### 21. How does a human loan officer actually interact with this system's output?

> "Never as an autonomous decision — always as a draft they review, edit if needed, and explicitly
> approve, reject, or send back for revision, the same DRAFTED-through-FINALIZED shape as course 2's
> sign-off workflow. In the canary and any broader rollout, the review UI shows the loan officer that the
> memo was AI-produced, which agents contributed, and — if any specialist agent's section is missing or
> flagged as incomplete because that agent failed — exactly what's missing, rather than a memo that reads
> as complete when it isn't. The officer can see each agent's individual findings, not just the final
> synthesized text, so if something in the memo looks off, they can trace it back to which agent produced
> it. Nothing about the system changes who has the authority to approve an underwriting decision, and
> nothing the system produces becomes part of the real record until a human has actually exercised that
> approval — the system's entire contribution is getting a well-organized, well-sourced draft in front of
> the loan officer faster than assembling one from scratch would take."

---

### Quick follow-up probes to be ready for

- "What's `output_config.effort` or the thinking-depth setting on Claude, and does it matter for this
  system?" → A capability specific to Claude's API surface with no direct Azure OpenAI equivalent —
  worth naming as one more concrete API-shape difference beyond the ones in Chapter 1, and a real,
  per-agent tuning lever: the Supervisor's routing judgment and the Memo Drafting Agent's synthesis are
  plausible candidates for a higher setting than a narrower, more mechanical agent like the Financial
  Spreading Agent.
- "How would you have caught the system-prompt formatting bug (Q7) even faster?" → A golden-request test
  in CI, run on every change to the shared adapter layer and checked against every agent's own payload
  shape individually, not just before the first big evaluation run — Chapter 7.
- "What's the difference between an agent failing and an agent's model being deprecated?" → A failure is
  transient and handled by the failure-isolation design (Chapter 7); a deprecation is a planned,
  vendor-announced event the per-agent scheduled re-validation check (Chapter 6) should catch well before
  it turns into a failure.
- "Is shadow mode the same thing as A/B testing?" → Related but not the same — A/B testing typically
  splits real users between two live variants that both affect real outcomes; shadow mode duplicates
  traffic to a path that never affects a real outcome at all, specifically because the system hadn't yet
  cleared the sign-off Chapter 4 requires before it's allowed to.
- "Why not just fine-tune the existing GPT deployment instead of building a whole multi-agent system?" →
  A reasonable alternative worth having considered — fine-tuning a single model can improve format/style
  adherence, but it doesn't change the underlying architectural fact that one model call has to hold four
  distinct kinds of analysis in one context at once; it doesn't buy failure isolation, parallelism, or an
  explicit escalation path the way decomposing into agents does, which is why this project specifically
  tested an orchestration-layer change rather than trying to fine-tune around a task-shape mismatch.
