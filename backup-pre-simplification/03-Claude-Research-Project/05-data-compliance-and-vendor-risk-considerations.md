# 05 — Data Compliance and Vendor Risk Considerations

## The fact this whole chapter follows from

Azure AI Foundry makes it technically trivial to reach Claude from inside an existing Azure environment
— a resource, a deployment, an API key, and every agent in the multi-agent system is calling Anthropic's
model within a few minutes of deciding to. It is tempting to read that technical ease as compliance ease
too. **It is not.** Even though the traffic never leaves Azure's network boundary and even though every
agent's calls sit behind the same Private Endpoints and Azure AD gate as everything else in this
environment (Chapter 4), calling Claude means data reaches **Anthropic**, a company with no prior vendor
relationship with this organization, for inference — for every one of the five agents, on every loan
application the system touches. That is a genuinely new third-party vendor relationship, layered on top
of — not replacing — the already-established Microsoft relationship the Azure OpenAI deployments run
under. "It's the same Azure environment" is a true and important statement about the network and
governance layer (Chapter 4); it is not a true statement about vendor risk, and treating those as the
same thing is the single mistake this chapter exists to prevent.

## Vendor risk assessment as a real gate, not paperwork

Introducing any new vendor that will process data — client data, employee data, or, in this system's
case, synthetic/de-identified data that's structurally shaped like real commercial loan applications
(financial statements, borrower profiles, credit policy queries) — triggers a vendor risk assessment:
security posture review, data handling practices, incident response commitments, subprocessor
disclosures, and contractual terms specific to this vendor. This isn't a box-checking exercise a project
team can route around by calling it "just a pilot" or "just agentic tooling" — it's a real control that
exists precisely to prevent exactly the failure mode of a new API integration quietly reaching real
applicant financial data before anyone with authority over vendor risk has reviewed the relationship.

For a project moving at the pace this one needs to move at, the honest and defensible position is:
**the vendor risk assessment for Anthropic is a real, necessary gate, and it takes real time — likely
longer than it takes to stand up the Azure AI Foundry resource itself.** A project that's serious about
compliance doesn't treat that timeline mismatch as an obstacle to route around; it treats it as the
reason development and evaluation start on data that doesn't need to wait for the assessment to
conclude. That's exactly the reasoning behind the next section, and it's worth stating in those terms if
asked — not "we skipped the vendor review because it was slow," but "we designed the rollout so it
didn't need the vendor review to be finished before useful work could start."

## Why the earliest phase runs on synthetic and de-identified data first — a strong answer, not a workaround

If asked directly, "how did you handle data compliance for a brand-new LLM vendor powering five
different agents," the strongest, most honest answer available is:

> "Development and the earliest evaluation started on synthetic loan-application data — generated to be
> structurally and statistically similar to real commercial-borrower financial statements, credit policy
> queries, and risk narratives (realistic financial-ratio distributions, plausible industry risk
> profiles, policy-corpus queries that mirror real covenant language) without containing any real
> borrower's actual information — and, once available, de-identified historical application data with
> direct identifiers removed or tokenized. It escalated to running the system on real, identifiable
> applicant data only after the vendor risk assessment for Anthropic concluded and a data processing
> agreement was in place. That wasn't a limitation we worked around — it was the only defensible order
> of operations for introducing a new AI vendor, powering a five-agent pipeline, to a regulated
> environment, and it meant the earliest, riskiest phase of the work (working out whether the
> orchestration design itself was sound, debugging the Azure AI Foundry integration across five separate
> agents, tuning the evaluation harness from Chapter 2) happened on data where a mistake had zero real
> consequence."

That's a genuinely strong answer to give in an interview, not a hedge — it demonstrates the candidate
understood that "can we technically call this API, from every agent in the pipeline" and "are we allowed
to send this data to this vendor" are two separate questions with two separate timelines, and designed
the rollout to make progress on the first without having to wait on, or worse, quietly skip, the second.
An engineer who can't articulate this distinction — who conflates "we have API access" with "we're
cleared to use it on real applicant data" — is exactly the failure mode a compliance-minded interviewer
is listening for.

The concrete synthetic-data design worth being able to describe: generate a set of fictitious commercial
borrowers with realistic financial-statement distributions (revenue, leverage ratios, industry
classification), synthetic credit-policy queries constructed to trip the same threshold and covenant
checks the Compliance Agent needs to exercise, and synthetic risk narratives covering the range of
industry-risk and macro-factor combinations the Risk Assessment Agent needs to reason over — including
the more complex, multi-factor cases that most directly test whether the Supervisor correctly escalates
ambiguous applications rather than letting the pipeline complete on thin evidence (Chapter 2).
`notebooks/01_comparative_eval_harness_demo.ipynb` and
`notebooks/02_context_window_chunking_comparison_demo.ipynb` both work against exactly this kind of
synthetic data — a pattern worth carrying into the real deployment, not just the notebooks.

## Data processing agreements: confirm, don't assume, no-training guarantees

The existing Azure OpenAI relationship almost certainly runs under a data processing agreement that
includes an explicit commitment: prompts and completions sent to the enterprise Azure OpenAI deployment
are **not used to train or improve the underlying model**, and Microsoft's standard Azure enterprise
terms typically carry data-handling commitments consistent with the compliance posture this whole
curriculum operates under. It is a real, common mistake to assume a new vendor accessed through the same
cloud marketplace automatically inherits an equivalent guarantee. **It does not, automatically, and the
correct posture is to confirm the specific terms in writing rather than assume them.**

Concretely, this means checking — as an explicit checkpoint in the vendor risk assessment, not an
afterthought — whether the specific commercial terms under which Claude is being accessed (whether
that's Anthropic's own enterprise terms, or terms specific to the Azure Marketplace/Foundry offering)
include a comparable no-training-on-customer-data commitment, what the data retention window is for
logged requests across all five agents (for abuse monitoring or debugging purposes, which is common and
reasonable, but should have a defined, short retention window rather than an indefinite one), and
whether those terms are consistent across the shadow-mode evaluation phase and the canary/production
phase (Chapter 4), or whether a different, more permissive tier applies during evaluation that would
need to be explicitly upgraded before real applicant data touches it in canary. None of this is a
hypothetical concern to wave away with "it's probably fine because it's the same cloud" — it's a
specific, checkable set of contractual facts, and the honest position going into this deployment is that
they had to be confirmed, not assumed, before any real applicant data was in scope for any agent.

## Why data residency gets *more* scrutiny for a new vendor, even though the traffic stays in-region

A natural, and wrong, intuition: "the Azure AI Foundry resource every agent calls is provisioned in the
same Azure region as everything else, so data residency is a solved problem, same as it always was." The
traffic staying in-region is a genuinely relevant fact, but it's not the whole picture for a new vendor
relationship, and here's why it deserves *more* scrutiny, not less, precisely because the environment
otherwise looks identical to what's already approved: a new vendor means new questions about **where
that vendor's own backend infrastructure and any subprocessors sit relative to the region the Azure
resource is provisioned in**, what happens to data in transit between the Azure Foundry front door and
Anthropic's serving infrastructure — for every agent's calls, not just one — and whether any support,
monitoring, or incident-response tooling on the vendor's side has access to logged request content from
outside the committed region. An already-trusted vendor relationship (Microsoft, for the existing Azure
OpenAI deployments) has already had these questions asked and answered as part of the original
platform-level compliance review. A brand-new vendor has not — and the surface-level similarity of "it's
provisioned in the same region, behind the same network boundary" is exactly what makes it easy to skip
asking the same questions again for the new vendor, which is the specific trap this section is naming.
The fact that everything *looks* the same architecturally is precisely why the compliance review can't
be waved through on architectural similarity alone.

## Tying it back

The throughline across this chapter: Azure AI Foundry solves the *technical* problem of reaching Claude
from inside an approved Azure environment cleanly and quickly, for every agent in a five-agent pipeline.
It does not, and cannot, solve the *vendor-relationship* problem of Anthropic being a genuinely new party
with access to applicant data, which needs its own risk assessment, its own confirmed contractual terms
(not inherited, not assumed), and its own data-residency scrutiny — even when every architectural fact
about where the traffic flows looks identical to what's already approved. Running the earliest phase on
synthetic and de-identified data until that review concluded wasn't a limitation to route around; it was
the correct, defensible sequencing for introducing any new AI vendor into a regulated environment, and
it's the strongest possible answer to give when a compliance-minded interviewer asks how a new vendor's
data handling was actually verified, not assumed, for a system now touching real credit decisions.
