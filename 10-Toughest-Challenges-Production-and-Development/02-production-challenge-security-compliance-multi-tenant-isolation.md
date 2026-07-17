# 02 — Production Challenge: Security, Compliance & Multi-Tenant Isolation

## Why this problem is close to unavoidable

The same platform team built production systems for **both HSBC and Bank of America** (Capco,
courses 1–4) and for **both Eli Lilly and AstraZeneca** (Indegene, courses 5–9). Any time one team
builds and operates similar systems for two direct competitors, the single hardest non-negotiable
requirement is isolation: **one client's data, prompts, documents, logs, and monitoring must never
be visible in the other client's environment** — not through a shared dashboard, not through a
shared log stream, not through a reused prompt template that leaks a client name, not through a
support engineer debugging the wrong tenant's ticket while looking at the wrong tenant's traces.
This chapter is the compliance-and-security-flavored answer to "toughest problem," and it pairs
naturally with an interview thread about regulated-industry experience.

## Two different compliance regimes, one shared discipline

| | Banking (HSBC / Bank of America) | Pharma (Eli Lilly / AstraZeneca) |
|---|---|---|
| Primary concern | Data confidentiality, network isolation, auditable access control | Content accuracy/audit trail, human review before publication, PHI-adjacent handling |
| Framework flavor | SR 11-7-style model governance, banking data-residency rules | GxP-style validation discipline, medical/legal/regulatory (MLR) review |
| Isolation mechanism | Azure AD tenant/subscription boundaries, VNet segmentation, Private Endpoints, per-client resource groups | Per-client AWS account or VPC boundary, IAM scoping, S3 bucket policies, Secrets Manager scoping |
| The "never" rule | HSBC traffic/logs must never appear in Bank of America's Azure Monitor workspace or vice versa | Eli Lilly content, prompts, or generated claims must never appear in AstraZeneca's S3 buckets, CloudWatch logs, or review queues |

### Banking-grade network and identity isolation

The Capco production stack described across courses 1–4 — Azure App Service behind Azure Front
Door/Application Gateway with WAF, VNet integration and Private Endpoints, Azure AD auth, Azure
Monitor/Application Insights — exists specifically to satisfy this kind of requirement:

- **Private Endpoints / VNet integration** ensure the application and its dependent Azure OpenAI
  deployment are not reachable over the public internet at all; traffic stays inside the client's
  virtual network, which also happens to be the mechanism that keeps HSBC's network path physically
  separate from Bank of America's.
- **Per-client Azure subscriptions or resource groups**, each with its own Azure AD tenant boundary,
  mean role-based access control (RBAC) can be scoped so that even platform-team engineers only have
  standing access to the client environment they're actively working in — least privilege applied
  at the tenant boundary, not just the application layer.
- **Azure Monitor / Application Insights workspaces are provisioned per client**, specifically so
  that a dashboard, log query, or alert built for one client's traffic can't accidentally surface
  the other's — a shared "global" monitoring view is a tempting convenience and a real risk if built
  carelessly.

### Pharma-grade content audit trail and human review

The Indegene stack (AWS ECS Fargate/ALB, Sagemaker, Lambda/API Gateway, S3, CloudWatch, Secrets
Manager) carries a different but equally strict discipline, centered on **content provenance and
human-in-the-loop review**:

- Any AI-generated claims content (e.g., from the GenAI Regulatory Platform in course 9, or claim
  extraction/tagging in course 7) needs a durable **audit trail**: what prompt/model version
  produced it, what source documents it was grounded in, who reviewed it, and when — because a
  pharma client's medical/legal/regulatory (MLR) review process is itself subject to regulatory
  audit.
- **Human-in-the-loop review before publishing** is a hard gate, not a nice-to-have: generated
  content (claims, summaries, structured regulatory documents) does not go to an end reader without
  a qualified human reviewer signing off, and the system needs to make that gate impossible to
  bypass accidentally.
- **PHI-adjacent data handling** — even when a system isn't processing patient-identifiable data
  directly, pharma content pipelines often sit close to clinical/patient-support content, so the
  same discipline (encryption at rest/in transit, least-privilege IAM, Secrets Manager rather than
  hardcoded credentials, S3 bucket policies scoped per client) is applied defensively.

## Where cross-tenant leakage risk actually creeps in

In practice, near-misses come from **shared infrastructure or shared code**, not from an engineer
directly copying one client's data into another's environment. The likely failure modes:

- A **shared prompt template or few-shot example library** that was originally written using one
  client's real content as an example and gets reused, unmodified, for the other client.
- A **shared logging/observability library** that defaults to verbose request/response logging,
  deployed identically into both client environments — fine in isolation, but risky if a debugging
  session ever pulls both clients' logs into the same terminal or ticket for comparison.
- A **shared internal tool** (an admin dashboard, a debugging notebook, a support runbook) built
  once for efficiency and pointed at "whichever client's environment" via a config flag — exactly
  the kind of convenience that, if misconfigured once, points at the wrong environment.

## Illustrative incident: a near-miss on a shared component

> **This is an illustrative, plausible scenario built from the isolation requirements above — not a
> confirmed account of an actual leakage event.** No real data exposure is being described; the
> point of this story is the *process* that catches this class of risk before it becomes real.

**Situation.** The platform team supporting both HSBC and Bank of America maintained a shared
internal library for building the LangChain/LCEL prompt pipelines used across both banks'
chatbot assistants (course 1), to avoid duplicating pipeline-construction code. During a routine
security review ahead of a new feature rollout, a reviewer flagged that the library's default
few-shot example set — bundled to give new pipelines a sensible starting point — had been seeded
using a real (if lightly redacted) example drawn from one bank's actual support ticket during
early prototyping, and had never been swapped out for a fully synthetic example before the library
was reused for the other bank's deployment.

**Task.** I needed to determine whether the real example had actually been shipped into the other
bank's production prompts (not just the shared repo), assess the actual exposure if so, and fix the
library so this class of mistake couldn't recur.

**Action.** I audited every deployed prompt configuration across both banks' environments to
confirm whether the flagged example had been included in any production system prompt — it had,
in a non-production staging environment for the second bank, but had not reached production traffic
because a config override had replaced it before go-live. I still treated this as a real near-miss:
I removed the artifact from the shared library entirely, replaced the default few-shot examples with
fully synthetic, clearly-labeled placeholder content, added a CI check that scanned any new prompt
template for known client-identifying strings before it could be merged, and wrote up the incident
for the model-risk/compliance stakeholders on both engagements so the process gap — not just the
one artifact — was documented and closed.

**Result.** *(Illustrative)* No real production exposure occurred, but the near-miss led to a
concrete process change: shared libraries used across clients now go through a "client-neutral
content" review step before merge, and the CI string-scan check has since caught two further
attempts to commit client-identifying example content, both in non-production branches, both
before merge.

## Talking points this chapter sets up

- Tenant isolation as a design discipline enforced at the infrastructure boundary (VNet/Private
  Endpoints, per-client Azure subscriptions, per-client AWS accounts/VPCs), not just application
  logic.
- The difference between banking's confidentiality-first isolation model and pharma's audit-trail
  and human-in-the-loop review model — and why both matter to this kind of platform team.
- Shared code/libraries as the realistic vector for cross-tenant risk, and CI-level guardrails as
  the fix that scales better than review-by-vigilance alone.
- Treating a caught near-miss as a reportable, process-improving event rather than something to
  quietly patch and move on from — a trait interviewers are specifically listening for.
