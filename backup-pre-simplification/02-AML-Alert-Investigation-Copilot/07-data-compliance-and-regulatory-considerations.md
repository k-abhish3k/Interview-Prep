# 07 — Data Compliance and Regulatory Considerations

## Why this chapter exists

AML/KYC and transaction data is among the most sensitive data a bank holds — it's simultaneously
financial data, personally identifiable information, and (because it exists specifically to document
suspicion of criminal activity) data whose mishandling carries its own distinct regulatory exposure on
top of ordinary data-protection obligations. This chapter treats data compliance as a first-class
design concern, not a bullet point at the end of an architecture review: data residency, PII
minimization in what actually reaches the model, access control, audit logging built for regulatory
examination, retention policy (including a genuinely counterintuitive point worth knowing), and vendor
risk.

## Data residency: why this runs inside the client's Azure tenant, not a public API

This reinforces, in a higher-stakes form, the same rationale Course 1 establishes for using Azure
OpenAI over the public OpenAI API: a deployment inside the client's own Azure OpenAI resource, in their
own tenant, keeps every prompt and every generated narrative inside the client's compliance boundary —
no AML/KYC/transaction data ever crosses into infrastructure the bank doesn't control. For this
project specifically, that's not a preference, it's close to a hard regulatory requirement: financial-
crime data is subject to strict jurisdictional handling rules in most regimes (data-localization
requirements in several jurisdictions a global bank operates in, and general expectations under
banking-secrecy and data-protection law that customer financial data doesn't transit third-party
infrastructure without explicit contractual and technical controls). Azure OpenAI's model is
specifically that the client's data stays within their Azure subscription/tenant boundary and inside
the same **VNet integration / Private Endpoints** topology as every other Capco course in this
curriculum — no backend service is reachable from the public internet, and the LLM calls themselves
never leave the private network path.

## PII minimization in prompts: a genuine design tradeoff, not a settled question

This is worth presenting as an actual decision with real tradeoffs on both sides, not a foregone
conclusion, because a candidate who's only prepared "always mask everything" or "the model needs full
context" hasn't actually thought about the tradeoff.

**The case for full, unmasked KYC/transaction data in the prompt:** the narrative's whole value is
precision — a case narrative that says "the customer" instead of the actual name, or a masked account
number instead of the real one, is materially less useful to the compliance officer who has to act on
it, and the officer will need the real values anyway to cross-reference other systems during review. If
the model only ever sees masked placeholders, its synthesis (comparing this customer's activity to
their own profile, correlating across prior cases) is working with less signal, and mistranslating a
placeholder back to the real value downstream is itself a new class of bug.

**The case for masked/tokenized PII in the prompt, with real values resolved only after generation:**
the LLM call itself is the one point in the pipeline where data leaves the direct control of the
bank's own database/application layer and passes through a third-party-operated (even if
tenant-isolated) model-serving layer. Minimizing what's actually sent — replacing account numbers,
full names, and other direct identifiers with stable tokens (`CUSTOMER_TOKEN_A417`,
`ACCOUNT_TOKEN_9B2`) before the prompt is assembled, generating the narrative against tokenized
placeholders, then resolving tokens back to real values in a post-processing step before the narrative
is stored or displayed — reduces the blast radius of any hypothetical logging, caching, or provider-
side issue to tokens that are meaningless outside this system, even though Azure OpenAI's own
enterprise terms (below) already commit to not retaining or training on this data.

A defensible middle position, and the one worth stating as the actual recommendation rather than
picking an extreme: **tokenize direct identifiers (name, account number, national ID) but leave
analytically meaningful values unmasked** (transaction amounts, dates, occupation category, risk
rating) — the model needs the *shape* of the data to reason well, but rarely needs the literal account
number to do so; the account number only becomes necessary once a human is verifying a specific cited
transaction against the source system, which happens after generation, at the point where tokens are
already resolved back for display. This mirrors the same instinct as Chapter 2's structured-vs-semantic
retrieval argument — send the model what it actually needs to reason well, not everything simply because
it's available.

## Access control: only the right people see a given alert's data

This ties directly to Course 5's RBAC/MSAL pattern (Course 5, Chapter 4) — the same Azure AD-app-role-
driven authorization model, applied here to a materially higher-stakes access decision. The specific
requirements this project adds on top of the general pattern:

- **Business-unit / customer-segment scoping.** An investigator or compliance officer should only see
  alerts (and the underlying KYC/transaction/case data behind them) for the customer segment or business
  line they're authorized to work — an AAD app role (or role claim) determines which alert queue a given
  signed-in user even sees, the same shape as Course 5's business-line routing, but gating access to
  sensitive financial-crime data rather than a document upload flow.
- **Case assignment as a second, narrower layer on top of role-based access.** Beyond "is this person an
  AML investigator," a specific alert is typically assigned to a specific investigator or a specific
  team's queue — access to a given alert's full case narrative, once generated, is scoped further to
  whoever it's assigned to (plus supervisory/compliance-officer roles with broader visibility), the same
  relationship-based-access idea Course 5 describes for its IWPB approver mapping (Course 5, Chapter 4),
  applied here as "assigned investigator" rather than "designated approver."
- **Read access to raw source data (KYC, transaction history) is logged identically to read access to
  the generated narrative.** A narrative is a synthesis, but the underlying source data is at least as
  sensitive — access-logging can't only cover the polished output and ignore the fact that generating it
  required querying the raw customer data underneath.

## Audit logging for regulatory examination readiness

This is the concrete requirement that a regulator can, at any point, request to see exactly what data
and what AI output informed a given compliance decision — and the system has to be able to answer that
completely, not approximately. Building on Chapter 4's audit-trail requirements (what was AI-drafted vs.
human-edited, by whom) and Chapter 5's data-snapshot recording, the full regulatory-examination-ready
log for any finalized narrative needs to reconstruct:

- **Exactly what data was retrieved/queried**, and when — the specific KYC snapshot, the specific
  transaction-history window and its parameters, the specific prior case notes retrieved and their
  relevance scores.
- **Exactly what the model generated**, verbatim, including which system-prompt version and which Azure
  OpenAI deployment/model version produced it — the same granularity Course 1's Application Insights
  tracing captures for chatbot responses (Course 1, Chapter 3), here applied to a document with direct
  compliance consequences rather than a conversational answer.
- **The full citation-verification and freshness-check results** at generation time and at
  approval time (Chapters 3 and 5) — not just whether the checks passed, but what they actually found.
- **Every human interaction with the narrative** — who viewed it, who edited it (with a diff against the
  AI draft, per Chapter 4), who approved or rejected it, and when.
- **Who accessed the underlying source data**, independent of whether they ever opened the generated
  narrative — access to raw KYC/transaction data is itself an auditable event, per the access-control
  section above.

The unifying design principle: **the audit log has to be reconstructable as a complete story, not a
scattered set of independent log lines** — a regulator's question is rarely "show me one log entry," it's
"walk me through exactly how this case was handled, start to finish," and the system needs to be able to
answer that as a coherent narrative of its own, built from data that was captured as a first-class
requirement rather than reconstructed after the fact from whatever logs happened to exist.

## Data retention policy: why "delete on request" is not simply available here

This is a genuinely non-obvious point worth including deliberately, because it cuts against the general
data-minimization instinct that governs most PII handling, and getting it right shows real domain
understanding rather than a generic "we follow GDPR" answer.

For most categories of personal data, a data-protection regime like GDPR (or equivalent frameworks)
gives individuals a right to request deletion, and general data-minimization practice favors deleting
data once it's no longer needed for its original purpose. **AML/financial-crime records are a
structural exception to that instinct**, not an oversight: AML regulation typically imposes **mandatory
multi-year retention requirements** on suspicious-activity records, case narratives, and the
investigative trail behind them — five years is a commonly cited baseline under several major regimes
(the exact figure varies by jurisdiction and record type, so treat any specific number as directional,
not a fact to cite as universal), specifically *so that* a regulator can examine historical AML decision-
making well after the fact. A bank that deleted case narratives on a standard data-minimization
timeline, or in response to an individual's deletion request, would very plausibly be **violating** its
regulatory retention obligation, not complying with a privacy right — the retention requirement
overrides the general deletion right for this specific category of record, a genuinely different answer
than the one that applies to most other PII the same bank holds.

This has concrete design consequences:

- **Retention periods are configured per-record-category, not globally** — a draft narrative that was
  rejected and never finalized plausibly has a shorter, more standard retention/deletion policy than an
  approved, finalized case record contributing to an actual AML determination, which needs the long,
  regulation-driven retention window.
- **The underlying source-data snapshot (Chapter 5) needs its own retention story**, separate from the
  narrative text itself — if a regulator examines a finalized case years later, "what did the customer's
  transaction history actually look like at the time" needs to still be answerable, which means the
  snapshot (or enough of it to reconstruct the narrative's grounding) has a retention obligation of its
  own, not just the narrative's prose.
- **A data-subject deletion request against underlying KYC/transaction data has to be evaluated against
  the AML retention obligation, not honored by default** — this is exactly the kind of nuance worth
  stating plainly in an interview: "delete on request" is the right default instinct for most PII this
  candidate's other projects handle, and it is specifically the wrong default here, which is worth
  knowing rather than assuming one policy fits every project in this curriculum.

## Vendor and model risk considerations

- **Azure OpenAI's enterprise terms** commit to the client's prompts and completions not being used to
  train or improve the underlying models, and not being reviewed by human moderators in the way the
  consumer-facing product may be — this is the specific contractual guarantee that makes routing
  sensitive AML data through a hosted LLM defensible at all, and it's worth being able to name that this
  guarantee exists rather than just asserting "Azure OpenAI is compliant" vaguely.
- **Model-version pinning and change control.** A regulated compliance-document-generation pipeline
  shouldn't silently pick up a new model version's behavior change without an explicit, tested change —
  the deployment is pinned to a specific model version, and any upgrade goes through the same
  Azure DevOps release-gate process (Course 6) as any other production change, with regression testing
  against the citation-verification and schema-compliance checks (Chapter 3) before the new version
  serves live narratives.
- **Vendor concentration risk**, worth naming as a real (if lower-priority) consideration: relying on a
  single LLM provider for a compliance-critical drafting function means an extended Azure OpenAI
  regional outage or a breaking API change is a real operational risk to the narrative-drafting
  capability — mitigated in the near term by the graceful-degradation posture in Chapter 6 (the system
  can flag alerts for manual narrative drafting rather than being fully blocked), not by a multi-vendor
  architecture, which would be disproportionate complexity for this project's actual risk profile.

## Tying It Back

Data compliance for this project isn't a checklist appended after the architecture is done — the tenant
boundary shapes where the LLM calls can run at all, the PII-minimization tradeoff shapes exactly what
reaches the model in the first place, access control determines who can even see an alert's data before
a narrative is ever generated, and the audit-trail requirement (built across Chapters 4, 5, and this
chapter) is what actually makes the system examination-ready rather than merely well-intentioned. The
retention point is the one most worth having ready as a "here's something non-obvious I understood about
this domain" answer: the instinct to minimize and delete data, correct almost everywhere else in this
curriculum, is specifically the wrong instinct for finalized AML case records, where mandatory
multi-year retention exists precisely so a regulator can examine the decision later — a real, deliberate
exception, not an oversight.
