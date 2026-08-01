# 99 — Interview Q&A: AML Alert Investigation Copilot

Read this last, after the chapters are fresh. **Q1 is a near-certain first follow-up** the moment an
interviewer understands what this system does — "why isn't this fully automated" is the natural
question for any GenAI system touching a regulated decision, and it's positioned first deliberately,
the same way Course 1 leads with its RAG-freshness question and Course 5 leads with the real,
actually-asked versioning question. **Q2 (data changes mid-review) and Q3 (data compliance) are
featured prominently right after it** — both are the kind of question a one-line answer doesn't
survive, and both map to this course's newest, most technically dense chapters (5 and 7).

---

### 1. Why isn't this fully automated? The system already drafts a complete narrative and cites its sources — why does a human still have to sign off on every single one?

This is the gotcha question, and it deserves the full worked answer from Chapter 4, not a one-line
deflection. The full version lives in `04-human-in-the-loop-compliance-sign-off.md`; here's the
compressed, interview-ready version:

> "The honest answer is that this isn't really a 'the model isn't good enough yet' problem — it's a
> regulatory accountability problem, and it's worth being precise about that distinction, because they
> lead to different designs. AML regulation places a legal obligation on the bank, and functionally on
> identifiable people in its compliance function, to exercise judgment about suspicious activity and be
> accountable for that judgment under regulatory examination. A regulator doesn't accept 'an AI system
> determined this was a false positive' as a substitute for a documented human determination — the
> framework assumes a human is answerable for the call, not a piece of software, and that's true
> independent of how good the model's drafts are. It's the same shape of argument as a hypothetically
> perfect calculator still not getting to sign a tax return.
>
> So the system is architected so a narrative literally cannot become part of the official case record,
> and an alert literally cannot close, escalate, or feed a Suspicious Activity Report, without an
> explicit, recorded compliance-officer approval. I built that as a real state machine — DRAFTED,
> UNDER_REVIEW, APPROVED or REJECTED, then FINALIZED, with a REGENERATE path on rejection — not just a
> UI convention, so it's structurally enforced rather than a policy someone could accidentally skip. And
> every approved narrative carries a full audit trail showing exactly what the model drafted versus what
> the officer edited, and who approved it, and when.
>
> The model's job in this architecture is to make that officer's review faster and better-informed —
> dense with the right facts, each one individually cited back to a specific KYC field, transaction ID,
> or prior case, so the officer has something concrete to verify rather than a well-written paragraph
> they have to take on faith. That's a very different, and much more defensible, framing than 'the AI
> decides and a human rubber-stamps it' — the review is a real gate with real teeth, evidenced by the
> fact that officers can and do reject drafts back for regeneration."

**Follow-ups to be ready for:**

- *"Isn't that just adding friction that defeats the point of automating this?"* No, and the reason is
  worth stating precisely: the automation target was never "remove the human," it was "remove the
  manual data-gathering and first-draft-writing burden," which Chapter 1 establishes is the actual
  bottleneck on documentation time, not the officer's judgment step — that was never the slow part.
  Speeding up drafting while keeping review intact is exactly the win the STAR summary's illustrative
  metrics describe.
- *"What happens if a compliance officer just rubber-stamps everything without really reading it?"*
  That's a real risk with any human-in-the-loop system, and it's a process/oversight problem more than a
  technology problem. The audit trail (Chapter 4) at least makes it detectable — an officer with a
  near-100% unedited-approval rate and unusually fast review times is a visible pattern a supervisor can
  investigate — but no system design fully prevents a human from failing to exercise real judgment.
  Worth saying honestly, rather than claiming the architecture solves a human-diligence problem it
  can't.
- *"Would you ever recommend removing the human step for the lowest-risk alerts?"* Worth answering
  carefully: even a "low-risk" alert can turn out, on investigation, to be the exception, and the
  regulatory-accountability argument (not a confidence-threshold argument) is what requires the human
  step. So no, not as a matter of routine policy — though a bank's own risk appetite and regulator
  relationship could in principle define a narrower fast-track review process for certain alert types.
  That's a policy decision for the client's compliance function to make, not an engineering call to make
  unilaterally in the system design.

---

### 2. What happens if new transactions arrive, or the customer's KYC profile changes, while their alert's narrative is being drafted or sitting in review?

This is the staleness question this course's Chapter 5 exists to answer in depth, and it's the direct
analog of the "source documents changed" question in Course 1 and the document-versioning question in
Course 5 — expect it as a near-automatic follow-up once an interviewer hears "RAG pipeline" and
"compliance review queue" in the same sentence.

> "Today, a narrative reflects a snapshot of the customer's KYC and transaction data as of the exact
> moment it was generated, and nothing automatically re-checks whether that snapshot is still current by
> the time a compliance officer actually signs off. That matters because review isn't instant — a
> narrative can sit in a queue for hours, or during a backlog, days — and a customer under active
> investigation is, almost by definition, someone whose account activity might keep evolving during
> exactly that window. So the realistic gap is: an officer could approve a narrative that accurately
> described the customer's data an hour ago, while missing a transaction that posted twenty minutes
> before they clicked approve.
>
> It's worth being precise about a distinction that's easy to conflate here: whether a narrative is
> internally coherent and well-grounded in the data it was generated from is a completely different
> question from whether that data is still current as of sign-off. I run a citation-verification check
> that answers the first question — does every cited transaction or case ID actually resolve against the
> data the narrative was built from. That check can pass perfectly on a narrative that's still gone
> stale, because it's checking grounding against the original snapshot, not currency against the
> customer's data right now. Those are genuinely orthogonal failure modes, and a review process that only
> checks one of them has a real gap.
>
> The design I'd put in place — and this is a proposal, not something I'm claiming shipped — is a
> lightweight freshness check run at two specific moments: when an officer opens the narrative for
> review, and again right before they submit an approval. It compares a content hash of the KYC profile
> and the highest transaction ID included in the original generation context against the customer's
> current data. If either has changed, the officer sees a hard, blocking prompt — 'customer data has
> changed since this draft was generated, regenerate before approving' — rather than a small icon easy to
> miss at the end of a long queue. They can still override it with a stated reason for genuinely trivial
> changes, but the override itself becomes a logged, explicit decision instead of a silent gap.
>
> And that ties directly back to the audit trail — the system needs to record not just who approved a
> narrative and when, but which exact data snapshot it was approved against, and whether the freshness
> check was clean or overridden. For a regulatory examination, 'was this decision made on current
> information' is very plausibly the actual question that gets asked, and the system needs to be able to
> answer it precisely, not just 'was this decision made.'"

**Follow-ups to be ready for:**

- *"Why not just re-run generation continuously for every narrative in the queue?"* Wasted work for the
  common case — most narratives sit untouched and unchanged — and it doesn't actually solve the
  problem: continuous regeneration still has a gap between the last regeneration and the actual approval
  click. Checking at exactly the two moments staleness becomes consequential (open, and submit) is
  cheaper, and closes the actual gap that matters.
- *"What's the cheapest thing you could do without the fingerprint-comparison design, to bound the
  damage today?"* A background sweep (hourly, not continuous) across the `UNDER_REVIEW` backlog,
  flagging anything untouched for more than some threshold (say, 4 hours) for a manual staleness spot
  check. Cheaper to build than the full open/submit check, catches the worst case (a narrative that's
  been sitting for days), and is the named hardening gap in Chapter 6 worth mentioning if pressed
  further.
- *"Is this the same problem as Course 1's RAG-freshness chapter?"* Related in spirit — both are "the
  world changed, and the system doesn't automatically notice" — but a different mechanism and a
  different stake: Course 1's problem is a shared retrieval index serving stale content to *every* user
  asking a related question; this is a *per-narrative* data snapshot going stale between generation and
  one specific approval decision. Worth drawing that distinction explicitly, rather than pattern-matching
  them into the same fix.

---

### 3. This system handles extremely sensitive AML/KYC/financial-crime data. Walk me through the data-compliance considerations, beyond just "we used Azure OpenAI."

Featured prominently because data compliance was called out as a first-class chapter in this course
(`07-data-compliance-and-regulatory-considerations.md`), not an afterthought — expect this question in
some form whenever the domain (AML, financial-crime, KYC) comes up.

> "A few layers, and I'd actually push back gently on 'we used Azure OpenAI' being a complete answer on
> its own, because it's necessary but not sufficient. Data residency is the foundation — everything runs
> inside the client's own Azure tenant, VNet-integrated with private endpoints, so no AML or KYC data
> ever transits infrastructure the bank doesn't control, which matters because financial-crime data is
> subject to strict jurisdictional handling rules in most regimes the client operates in.
>
> On top of that, there's a genuine design tradeoff worth naming rather than glossing over: how much raw
> PII should actually reach the model in the prompt. Full unmasked data gives the model the most signal
> to reason well, and the officer needs real values anyway to act on the narrative. But every token that
> reaches the model is a token that's left the bank's own database and application layer, even inside a
> tenant-isolated environment. The position I'd actually defend is a middle one — tokenize direct
> identifiers like account numbers and names before they hit the prompt, resolve them back to real values
> only after generation, but leave analytically meaningful fields like transaction amounts and risk
> ratings unmasked, since the model needs the shape of the data to reason well and rarely needs the
> literal account number to do it.
>
> Access control ties to the same RBAC pattern Course 5 built for its document-uploader service — Azure
> AD app roles scope which business line or customer segment's alerts a given investigator even sees, and
> case assignment is a second, narrower layer on top of that, the same relationship-based-access idea
> Course 5 uses for its approver mapping.
>
> Audit logging has to be built for a specific, high bar: a regulator can request to see exactly what
> data and what AI output informed a given compliance decision, and the system has to reconstruct that as
> a complete story — what was retrieved, what the model generated, what was edited, who approved it, and
> against what data snapshot — not just produce scattered log lines.
>
> And there's one genuinely counterintuitive point I think is worth knowing here: the usual instinct with
> PII is minimize and delete when no longer needed, and a data-protection regime typically gives
> individuals a right to request deletion. AML records are a real, structural exception to that — AML
> regulation imposes mandatory multi-year retention on suspicious-activity records specifically so a
> regulator can examine historical decision-making later, and a bank that deleted case narratives on a
> standard privacy-driven timeline would very plausibly be violating its retention obligation, not
> honoring a privacy right. That's a genuinely different answer than the one that applies to most other
> PII, and getting that right is a real compliance judgment call, not just an engineering default."

**Follow-ups to be ready for:**

- *"How would you decide what to tokenize versus leave unmasked, concretely?"* Direct identifiers
  (name, account number, national ID, address) get tokenized; anything the narrative's synthesis
  genuinely needs to reason about (amounts, dates, categories, risk tiers, transaction counts) stays
  unmasked. The test is: does the model need this value's actual content to do its job, or just a
  stable reference to it.
- *"What does Azure OpenAI's enterprise commitment actually guarantee here?"* That prompts and
  completions aren't used to train or improve the underlying models, and aren't reviewed by human
  moderators the way a consumer product might be — the specific contractual guarantee that makes routing
  this data through a hosted LLM defensible, worth being able to name rather than asserting vaguely.
- *"If a customer submitted a GDPR-style deletion request, and their KYC data is part of an approved
  SAR-contributing case narrative, what happens?"* The AML mandatory-retention obligation overrides the
  general deletion right for that specific record category — the deletion request would be evaluated
  against, and very likely denied for, the finalized case record and its underlying data snapshot,
  though it might still apply cleanly to unrelated data the bank holds about that same customer outside
  the AML case file. This is exactly the nuance worth stating plainly, rather than defaulting to "always
  honor deletion requests."

---

### 4. (Behavioral) Tell me about this project.

Use the STAR summary from `00-README.md`, practiced to under 90 seconds:

> At Capco, a global banking client's AML investigation team was spending a large share of each
> investigator's day manually assembling case narratives for transaction-monitoring alerts — pulling KYC
> data, transaction history, and prior case notes from separate systems and writing them up by hand for
> every alert, most of which turned out to be false positives. I was asked to architect a GenAI copilot
> that could auto-draft those narratives while keeping a compliance officer firmly in the loop for
> sign-off, since a wrong AML determination has real regulatory and financial-crime consequences that
> make full automation inappropriate. I built a RAG pipeline that used two different retrieval
> mechanisms deliberately — a structured, auditable query tool for KYC and transaction data, and Azure
> AI Search's semantic retrieval for prior case notes, the one source that's genuinely unstructured
> text. I used structured-output prompting against a six-section narrative template with a per-claim
> citation requirement as the primary anti-hallucination defense, built the review workflow as a real
> state machine with a full audit trail, and designed a freshness check that flags a narrative for
> regeneration if the customer's data changed after drafting but before sign-off. It ran on the client's
> existing Azure production stack — Azure OpenAI, Azure AI Search, App Service, VNet/Private Endpoints,
> Azure AD — the same platform pattern as the AI Chatbot Assistant project. *(Illustrative)* It cut
> per-alert documentation time by roughly half and improved alert-to-closure turnaround by a comparable
> margin, while every finalized narrative still carried an explicit compliance-officer approval.

Remember to swap the italicized result for real numbers, and to say "a global banking client" rather
than naming a specific bank, unless you've checked your NDA and are confident it's accurate.

---

### 5. Walk me through what happens, end to end, from an alert firing to a case being closed.

1. The transaction-monitoring system fires an alert.
2. The data-gathering layer pulls KYC via a structured query tool, transaction history via the same
   structured tool (with an explicit, alert-type-appropriate lookback window), and retrieves prior case
   notes via Azure AI Search semantic/hybrid search.
3. All three feed one narrative-generation prompt, tagged by source.
4. Azure OpenAI generates a structured six-section narrative with per-claim citations and per-section
   confidence scores.
5. A citation-verification check confirms every citation resolves against the actual retrieved data,
   flagging any that don't.
6. The narrative enters the `DRAFTED` state and lands in a compliance officer's review queue.
7. The officer opens it (triggering a freshness check), reviews it (using the confidence indicators to
   prioritize attention), and edits/approves or rejects it (triggering a second freshness check at
   submission).
8. An approval moves the narrative to `APPROVED`, then automatically to `FINALIZED`, writing the
   immutable case record and, if warranted, notifying the SAR filing system.
9. A rejection triggers a `REGENERATE` cycle, retaining the original rejected draft.

See `00-README.md`'s architecture diagram and Chapters 2–5 for the full detail on each stage.

---

### 6. Why did you split retrieval into a structured query tool for KYC/transactions and Azure AI Search for prior case notes, instead of putting everything in one vector index?

Chapter 2's core argument: KYC and transaction data are structured, exact-match data, where a direct,
parameterized query is deterministic, cheaper, faster, and — critically for a compliance document —
trivially auditable (you can point to the exact query and the exact row it returned). Embedding-based
retrieval is a fuzzy, best-effort mechanism by construction — appropriate when the question is "find me
content that's semantically similar to this," inappropriate when the question is "what is this
customer's exact risk rating," which has one deterministically correct answer a vector index can only
approximate.

Prior case notes are the opposite case: free text written by different investigators, in different
words, describing similar underlying concerns — exactly the problem semantic retrieval is built to
solve, and a structured query cannot, since there's no schema to query your way to "notes that describe
a similar situation in different phrasing." The one-line version: match the retrieval mechanism to the
actual shape of the data, don't default to a vector index because a RAG pipeline already exists for one
source.

---

### 7. How do you actually enforce that a case narrative always has all six required sections, in the right structure?

Structured-output/function-calling mode against a Pydantic schema (Chapter 3) — the same pattern Course
1's Chapter 1 covers in depth for reducing format drift. Rather than prompt-engineered "please respond
in this JSON format" instructions that degrade under probabilistic compliance, Azure OpenAI's native
structured-output mode constrains generation to match the schema at the API level. A missing or
malformed section becomes a schema-validation failure the orchestration layer catches immediately,
before the narrative ever reaches a reviewer, rather than a subtle formatting slip a compliance officer
has to notice themselves.

One deliberate schema choice worth mentioning: each section is a list of individually-cited claims
rather than one prose paragraph, which forces a citation (or an honest admission it can't be cited) onto
each discrete assertion, instead of letting one well-cited sentence implicitly cover for uncited
neighbors in the same paragraph.

---

### 8. What's the primary anti-hallucination defense in this system, and how is it different from just telling the model to "only answer from the provided context"?

The per-claim citation requirement (Chapter 3), and the key difference is *verifiability, after the
fact*. "Only answer from provided context" (Course 1's RAG-grounding approach) is a prompt-level
instruction — it reduces hallucination, but there's no independent way to check whether a specific
generated sentence actually came from the context or not; you're trusting the instruction worked.

Requiring every factual claim to cite a specific KYC field, transaction ID, or prior case ID converts
that trust into something mechanically checkable: a post-generation verification step confirms every
cited ID actually resolves against the data that was genuinely retrieved/queried for that narrative. A
citation that doesn't resolve is a hallucinated citation — caught automatically, before a human ever has
to notice it by cross-referencing manually.

Worth being precise about the boundary of what this catches, though: it verifies a citation *points to
something real*, not that the claim built on it is fully accurate or appropriately weighted — that
harder judgment call is exactly why the human review step in Chapter 4 still exists, even when every
citation checks out cleanly.

---

### 9. What would you do if the citation-verification check found a hallucinated citation in a narrative — block it, or let it through to review?

The system's actual behavior (Chapter 6's error-handling table): it doesn't block generation outright,
because that would deny the officer a mostly-good draft over one bad claim, and it doesn't silently drop
the offending claim either, because that just relocates the problem rather than fixing it. It routes the
narrative to review with the specific unresolved citation flagged prominently, so the reviewing officer
knows exactly which claim to scrutinize rather than trusting the whole document uniformly. The more
conservative, transparent option is generally the right default for a compliance document — visibility
over the flagged issue, not silent handling either direction.

---

### 10. (System design) A batch transaction-monitoring run just fired 500 new alerts simultaneously. Walk me through how the narrative-generation pipeline handles that.

Chapter 6's burst-handling design: newly-triggered alerts are enqueued rather than fanned out as 500
simultaneous Azure OpenAI calls, since that would immediately blow through the deployment's TPM/RPM
quota. A worker pool drains the queue at a rate tuned to stay under that ceiling with headroom — the
same backoff/circuit-breaker discipline Course 1's Chapter 7 covers (exponential backoff with jitter, a
narrow allowlist of retryable status codes, respecting `Retry-After`), applied here to a
queue-draining worker pool instead of a single conversational call.

The queue isn't strict FIFO — alerts the upstream monitoring system already scores as higher-severity
are prioritized ahead of routine ones, so a burst doesn't bury an urgent case behind hundreds of routine
ones. Investigators see queue position and estimated wait for a pending narrative, rather than the alert
simply not appearing — the same "never leave the user guessing" instinct Chapter 6 applies to
backpressure as it applies to degraded data.

A circuit breaker matters more here than in a single-conversation chatbot specifically: if the burst
coincides with an Azure OpenAI degradation, letting hundreds of independently-retrying workers each
hammer the same struggling dependency is a materially worse outcome than tripping a breaker and pausing
new calls for a cooldown window once a failure threshold is hit.

---

### 11. Which failures should block narrative generation, and which should just degrade the narrative gracefully? How did you decide?

Chapter 6's error-handling table, and the underlying rule worth stating explicitly: the line is drawn on
whether the failed source is *load-bearing to the narrative's substance*, not on whether it's structured
or unstructured data.

- A KYC lookup failure or a transaction-history query failure **blocks generation entirely** — customer
  identity and the transaction pattern that triggered the alert are load-bearing for every other
  section, and a narrative built without them isn't a degraded narrative, it's a narrative about the
  wrong (or no) baseline.
- A prior-case-notes retrieval failure (Azure AI Search) **does not block** — it generates with that
  section explicitly marked as unavailable due to a retrieval error and a low confidence score, because
  prior context is valuable but not load-bearing, and most alerts legitimately have thin or empty
  prior-case sections anyway.

The other design rule worth naming: every degraded section carries an explicit, visible gap marker — a
system distinguishing "we looked and found nothing" from "we couldn't look" matters, because conflating
them either false-alarms on normal empty results, or silently hides a real outage.

---

### 12. Tell me about a bug you found (or would expect to find) that's specific to this domain, not just generic backend work.

Pick one of the three from Chapter 6 — the hallucinated-citation one is the strongest choice, because
it's a genuinely compliance-specific failure with a genuinely compliance-specific detection mechanism.

An early citation instruction told the model to "cite the relevant transaction ID" without explicitly
constraining it to IDs actually present in the supplied context. On narratives involving unusually large
transaction histories, the model occasionally cited a plausible-looking but non-existent transaction ID
— right format, right customer, wrong actual ID — rather than one genuinely in the retrieved set.
Nothing about this threw an exception or looked wrong on a quick read; it's exactly the "looks fine,
isn't" class of bug. It was caught by the citation-verification check itself once it was run
systematically against live narratives, tracing back to the prompt's implicit rather than explicit
scoping.

What would have caught it earlier: running that verification check against a held-out evaluation set
*before* launch, not only reactively once a live pattern showed up — the control that eventually caught
it already existed as a Chapter 3 design, it just wasn't exercised early enough in the development
lifecycle.

---

### 13. How is this project different, architecturally, from Course 1's AI Chatbot Assistant, even though both are Azure OpenAI + Azure AI Search RAG systems?

Several genuine differences worth naming, not just "same stack, different domain":

1. Course 1 is conversational — multi-turn, streaming, designed for back-and-forth. This project's
   generation step is a single, well-specified batch call producing one structured document, closer to a
   report-generation pipeline than a chat.
2. Course 1 retrieves everything through one RAG mechanism (Azure AI Search). This project deliberately
   splits retrieval two ways — a structured query tool for KYC/transactions, Azure AI Search only for
   prior case notes — because two of its three sources are structured data a vector index would serve
   worse than a direct query (Chapter 2).
3. Course 1's output is a conversational answer with a citation as a nice-to-have. This project's output
   is a compliance document where per-claim citation is a hard requirement and the primary
   anti-hallucination control, not a UX enhancement.
4. Course 1 has no mandatory human-approval gate before an answer reaches the user — a bad answer is
   recoverable with a follow-up question. This project structurally cannot let a narrative become a
   decision without a human sign-off, because the decision it feeds is high-stakes and effectively
   irreversible once acted on.

The shared foundation (Azure OpenAI, Azure AI Search, structured-output patterns, the production Azure
topology) is real and worth citing as reuse across the two projects — but the differences are exactly
where the more interesting engineering judgment lives.

---

### 14. What would you change if you rebuilt this today?

A few concrete things, in the same spirit as Course 1's answer to the same question:

1. I'd build the citation-verification check as a pre-launch evaluation gate from day one — run it
   against a held-out set of alerts before the system ever sees live traffic — rather than treating it
   as a production monitoring control that catches issues reactively, which is exactly the gap the
   hallucinated-citation bug (Q12) exposed.
2. I'd make the data-freshness check (Chapter 5) include the proactive backlog sweep from the start, not
   just the two point-in-time checks — the gap where a narrative can go stale for days without anyone
   noticing until they happen to open it is a real one worth closing from day one, rather than as a
   later hardening pass.
3. I'd design the PII-tokenization approach (Chapter 7) into the prompt-assembly layer from the
   beginning rather than deciding it as a retrofit — it's a much cleaner design to build
   token-resolution as a first-class step in the pipeline than to bolt it onto an existing full-PII
   prompt path later.
4. I'd lean on an LLM-evaluation harness in the shape Course 4 covers (faithfulness, completeness,
   hallucination-rate tracking) from the start, specifically scored against this project's own
   citation-resolution metric, rather than treating evaluation as a separate later initiative — for a
   compliance-facing system, "how do we know a prompt or model change didn't regress narrative quality"
   should be answerable from day one, not bolted on after the fact.

---

### 15. How would you evaluate this system's quality before and after a prompt or model change?

Beyond the citation-resolution check (a strong, cheap, automatable proxy for grounding quality — Chapter
3), a proper evaluation harness in the shape Course 4 (Model Risk Monitoring) establishes would run a
held-out set of representative alerts (ideally spanning multiple alert typologies and risk levels)
through the pipeline and score:

- **Citation-resolution rate** — what fraction of citations resolve cleanly.
- **Section-completeness** — does every generated narrative have non-empty, substantive content in each
  required section, or does it degrade into thin placeholder text.
- **Agreement with human-reviewed outcomes on a sample** — the harder, more valuable metric: how often
  an officer needed to make a material edit (not just a stylistic one) versus approve as-is, as a proxy
  for how much the draft is actually reducing the officer's real work versus just producing
  plausible-looking text that still needs substantial rework.

Faithfulness/hallucination-style LLM-as-judge scoring (Ragas-style, per Course 4) is a reasonable
complement for catching subtler issues the mechanical citation check can't, like a claim that's
technically well-cited but overstates what the cited data actually shows.

---

### 16. Why does this system use two separate states for "approved" and "finalized" instead of just one?

Chapter 4's design: `APPROVED` is the human action — an officer clicking approve, potentially after
edits — and `FINALIZED` is the system action that follows it: writing the immutable case record and, if
applicable, notifying downstream systems like the SAR filing queue.

Keeping them as two distinct, separately-timestamped states means an approval decision and its
downstream system consequences are two independently auditable events. That matters if a system-side
failure (say, the finalization write fails after an officer has already approved) ever needs to be
distinguished from the human decision itself in an audit trail — you can prove the officer approved
even if the system-side finalization step had a separate, later problem, rather than conflating "the
human decided" with "the system successfully recorded it" into one ambiguous state.

---

### 17. How does the "confidence" score attached to each narrative section actually work — is it the model's own certainty?

Deliberately not framed as the model's linguistic confidence (Chapter 3) — a section can read
grammatically assertive while resting on thin evidence, and that's exactly the gap this score is meant
to expose rather than mask. It's instructed to reflect how well the retrieved/queried context actually
supports that section's content. For instance, a historical-context section built from zero retrieved
prior cases should score low confidence even if the model still writes a fluent sentence about it,
because there's genuinely little grounding behind it.

That score becomes the input to review prioritization (Chapter 4) — an officer's attention is drawn to
weakly-grounded sections first rather than reading uniformly, which matters given real caseload pressure
and limited review time per alert.

---

### 18. What's the relationship between this project and Course 5's Document Uploader Service, if any?

No direct pipeline relationship claimed — they're different Capco engagements, potentially for
different business lines even at the same client — but two patterns transfer directly and are worth
citing as reuse:

- The RBAC/MSAL access-control pattern (Course 5, Chapter 4) applies here to scoping which investigator
  sees which alert's data.
- The relationship-based-access idea Course 5 uses for its IWPB approver mapping (an email match against
  an approver table, independent of role-based access) maps cleanly onto case assignment here — an
  investigator's access to a specific alert's narrative is scoped by assignment, a narrower layer sitting
  on top of the broader role-based access that determines which business line's alerts they see at all.

Worth naming both explicitly if asked how this project reuses patterns established elsewhere in the
curriculum, rather than presenting every project as built from scratch.

---

### Quick follow-up probes to be ready for

- "What's the difference between the citation-verification check and the freshness check?" → the first
  checks grounding against the *original* snapshot (Chapter 3); the second checks currency against
  *current* data (Chapter 5) — orthogonal, both can independently pass or fail, see Q2.
- "Why not embed the KYC record too, just in case it helps retrieval?" → embedding a record with one
  correct answer per field trades away determinism and auditability for a retrieval mechanism it doesn't
  need — Chapter 2.
- "What's the single most important control against a wrong AML determination in this whole system?" →
  arguably neither citation verification nor freshness checking alone, but the combination plus the
  non-negotiable human sign-off (Chapter 4) — no single automated control is treated as sufficient on its
  own for a decision this consequential.
- "How would you explain to a skeptical compliance officer why they should trust a draft from an LLM at
  all?" → don't ask them to trust it blindly — every claim is individually citable back to a specific,
  verifiable source, which is a fundamentally different trust model than "trust the model's synthesis,"
  and the review workflow is designed around verification, not blind acceptance.
- "What's illustrative versus confirmed in this course?" → everything is an illustrative, technically
  detailed reconstruction — there's no source repository behind this project the way Course 5 was rebuilt
  from real code — treat quantified metrics and specific implementation details as defensible, replaceable
  placeholders, not verified facts, per the note in `00-README.md`.
