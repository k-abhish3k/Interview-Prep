# 07 — Development Challenge: Integration with Legacy Client Systems

## Why this is the most "consulting-specific" challenge in the folder

Building a modern, cloud-native GenAI service is one problem. Getting it to actually talk to a
Fortune-500 bank's or global pharma company's existing systems — and getting permission to do so —
is a different and often larger problem, one that's easy to underestimate before you've lived
through it. Both Capco and Indegene engagements sit on top of client organizations with decades of
existing IT investment, layered security review processes, and procurement cycles that don't move
at startup speed. This chapter is the answer for "tell me about a frustrating technical problem
that had nothing to do with the actual algorithm" — a question that separates candidates who've
only worked in greenfield environments from those who've shipped inside a large regulated
enterprise.

## Three distinct flavors of legacy-integration pain

### 1. Legacy protocols and on-prem APIs

A bank's policy-document system, ticketing platform, or core-banking-adjacent data source is
frequently not a modern REST API — it's a SOAP service, a mainframe-adjacent system with a narrow
batch-file or FTP interface, or an on-prem system reachable only from inside the bank's own network,
not from a cloud service without a dedicated network path. Building a RAG pipeline (course 1) or a
document-ingestion service (course 5) against a source like this means writing and maintaining
adapter code for protocols and data shapes that have no modern client library, often with sparse or
outdated documentation, and no ability to change the legacy side to make integration easier — the
legacy system predates the project and won't be modernized for it.

### 2. Enterprise network and security review cycles

Before a new service is allowed to call *out* of the client's network to Azure OpenAI, or before an
AWS-hosted service is allowed to call *into* a pharma client's on-prem document store, the client's
security team runs its own review: firewall rule requests, proxy/allowlist configuration, penetration
testing of the new integration point, and sign-off from a security architecture board — none of
which the engineering team controls the timeline for. These reviews exist for good reason (the same
reason VNet/Private Endpoint isolation exists in the production architecture — chapter 02), but they
routinely take weeks, run on their own review-board cadence rather than on-demand, and can bounce a
request back for rework if the original request didn't anticipate a requirement (e.g., the initial
firewall rule request scoped too broad an IP range, or didn't account for a specific data-classification
label the security team required on the traffic).

### 3. Procurement and approval gates

Provisioning a new Azure OpenAI deployment, a new Sagemaker endpoint, or even a new service account
inside the client's environment can itself require a formal change request, a business-justification
document, and sign-off from a client stakeholder who isn't on the project day to day and has their
own queue of approvals to work through. This is invisible in a personal or startup project and can
become the single largest source of calendar time on a consulting engagement's timeline — a
two-day engineering task blocked for three weeks on an approval that has nothing to do with the
engineering itself.

## What actually works against this class of problem

- **Front-load the discovery.** Identify every legacy system and every network/security boundary
  the project will need to cross as early as possible — ideally during the estimation/scoping phase,
  not after development starts — because the lead time on a security review or procurement approval
  is usually the true critical path, not the engineering work.
- **Submit the broadest reasonable request the first time.** A security/network review round-trip is
  expensive in calendar time, so scoping the initial request too narrowly (and needing a second round
  for something you should have anticipated) is a common, avoidable mistake — better to ask once for
  everything the architecture will plausibly need, clearly justified, than to ask three times.
- **Build an abstraction layer around the legacy integration**, so the messy adapter code (SOAP
  client, legacy auth flow, brittle field-mapping) is isolated behind a clean internal interface —
  this both makes the rest of the codebase sane to work in and means a future legacy-system
  replacement doesn't require touching the rest of the application.
- **Run the approval/procurement track in parallel with engineering, not after it.** Submitting the
  network-access or provisioning request the moment the architecture is settled — rather than
  waiting until the code is ready to deploy — turns a serial multi-week bottleneck into overlapped
  work.

## Illustrative struggle: a network review cycle nearly cost a launch date

> **Illustrative, plausible scenario — not a verified account of a specific engagement's actual
> timeline.** Built from the realistic shape of enterprise network-review and procurement processes
> described above.

**Situation.** During development of a document-uploader/ingest service (course 5) for a banking
client, the service needed to pull source documents from an existing on-prem document-management
system reachable only from inside the bank's corporate network, while the new service itself was
being deployed on Azure App Service with VNet integration. The integration required a new,
specifically-scoped firewall rule and a security review of the new network path before any real
traffic could flow, and the review request wasn't submitted until the application code was already
functionally complete and ready to test against real data — because the network requirement hadn't
been fully scoped during initial planning.

**Task.** With a client-committed launch date a few weeks out, I needed to get the network path
approved and working in time, without simply accepting a slipped date, and without cutting a corner
on the security review that a slipped date might otherwise have tempted.

**Action.** I escalated the request immediately with a complete, specific justification package
(exact source/destination, ports, protocol, data classification, expected traffic volume) rather
than a vague ask, specifically to avoid a second review round-trip if the initial request was judged
incomplete. In parallel, since the actual document flow through the new network path couldn't be
tested until the firewall rule was live, I built and tested the ingest service against a
representative mocked version of the legacy document-management interface, so that once the network
path was approved, the only remaining work was an integration smoke test rather than first-time
development against the real system under time pressure. I also flagged the risk to the account/
delivery lead early rather than waiting to see if the review would clear in time on its own, so the
client side had visibility and could apply their own internal escalation channel if needed.

**Result.** *(Illustrative)* The network review cleared with about a week of buffer before the
committed launch date, in part because the complete first-pass request avoided a second review
round-trip, and the pre-built mocked-interface testing meant the remaining integration work after
approval took days rather than the multi-week effort it would have taken starting from scratch.
Scoping legacy network/security requirements during initial planning — rather than after development
was underway — became a standing checklist item for subsequent engagements on the same account.

## Talking points this chapter sets up

- The distinction between engineering critical path and procurement/security-review critical path —
  and why the latter often dominates on a large regulated-enterprise engagement.
- Front-loading discovery of legacy/network dependencies during scoping, not after development
  starts.
- Building against a mocked interface to de-risk the engineering work while an external approval is
  in flight, rather than serializing the two.
- Escalating and flagging risk early to non-engineering stakeholders as a deliberate practice, not a
  last resort.
