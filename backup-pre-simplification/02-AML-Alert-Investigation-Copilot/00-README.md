# 02 — AML Alert Investigation Copilot

> Resume bullet (verbatim): *"AML Alert Investigation Copilot: Architected a GenAI copilot for a global
> banking client that auto-drafts investigator-ready case narratives for AML/transaction-monitoring
> alerts — retrieving and synthesizing KYC profiles, transaction history, and prior case notes via a
> RAG pipeline (Azure OpenAI + Azure AI Search) — cutting analyst documentation time and accelerating
> alert-to-closure turnaround while keeping a compliance officer in the loop for sign-off."*

## The load-bearing fact of this whole course, stated up front

**A compliance officer signs off on every narrative before it becomes part of the official case
record. This system does not, and by design cannot, close an AML alert on its own.** Everything else
in this course — the RAG architecture, the structured-drafting prompt engineering, the citation
requirements, the staleness checks — exists in service of that one constraint. A wrong AML
determination has real regulatory and financial-crime consequences (a missed Suspicious Activity
Report is a genuine financial-crime failure; a wasted investigation misdirects a scarce, expensive
human investigator), so the interesting engineering problem here isn't "can an LLM write a
convincing case narrative" — it clearly can — it's "how do you build a system that drafts
*extremely well* while making it structurally impossible for that draft to become a decision without
an accountable human reading it first." Chapter 4 covers the review workflow in depth; treat this
paragraph as the answer to give if an interviewer only has time for one question about this project.

## Business Context

Banks run rule-based (and increasingly ML-assisted) transaction-monitoring systems that watch account
activity for patterns associated with money laundering — structuring (breaking a large transaction
into many smaller ones to stay under a reporting threshold), rapid movement of funds through multiple
accounts, activity inconsistent with a customer's stated occupation or expected transaction profile,
and dozens of other rule families. When a pattern trips a rule, the system raises an **alert**.

The well-known, industry-wide problem with these systems is a **very high false-positive rate** —
often cited in the 90%+ range across the industry, though the real figure varies enormously by bank,
rule tuning, and customer segment. Every alert, true positive or not, has to be reviewed by a human
AML investigator, who pulls together the customer's KYC (Know Your Customer) profile, the transaction
history that triggered the alert, and any prior case notes on that customer, and writes a **case
narrative**: a structured document that summarizes what happened, compares it to the customer's
expected behavior, calls out red flags, and recommends an outcome — close as false positive, escalate
for deeper review, or (in the most serious cases) contribute to a **Suspicious Activity Report (SAR)**
filed with the relevant financial-crime regulator (FinCEN in the U.S., the NCA in the UK, and
equivalent bodies elsewhere).

Writing that narrative well — pulling accurate data from multiple systems, synthesizing it into a
coherent, well-organized document, under a caseload that can run into dozens of alerts a day per
investigator — is slow, repetitive, and exactly the kind of multi-source-synthesis task GenAI is good
at. It is also, simultaneously, a **high-stakes, regulated decision domain** where full automation is
inappropriate: a hallucinated fact in a narrative, or a subtly wrong comparison to "expected behavior,"
could drive a real compliance officer toward the wrong call. That tension — strong GenAI fit on the
drafting side, genuinely risky on the decision side — is why this project exists as a **copilot**, not
an autonomous decisioning system, and it's the thread every chapter in this course pulls on.

## Client & Production Framing

The resume bullet names the client generically as **"a global banking client"** — it does not name a
specific bank. This course frames it plausibly as **HSBC**, consistent with Capco's established
primary banking client throughout this curriculum (see the root [`README.md`](../README.md)'s
"Client & Production Context" section), and consistent with this being an adjacent, sibling system to
Course 1's AI Chatbot Assistant — the two projects plausibly share the same Azure platform footprint
at the same client. **But say so honestly**: the resume bullet itself only commits to "a global
banking client." Use "HSBC" in an interview only if you specifically recall that being accurate for
this engagement (and have checked your NDA per the root README's confidentiality note); otherwise
"a global banking client" is both accurate to the bullet and safe to say regardless of what you
recall.

This went to **production with a customer-facing interface** — in this case, "customer-facing" means
investigator-facing and compliance-officer-facing: the people using this system daily are internal
AML analysts and compliance officers, not retail bank customers, but it is a real, live, production
tool serving real daily alert volume, not a proof-of-concept. It ran on the same Azure production
topology as every other Capco course in this curriculum: **Azure OpenAI**, **Azure AI Search**,
**Azure App Service** (with deployment slots), **Azure Front Door / Application Gateway** (WAF + TLS
termination) at the edge, **VNet integration / Private Endpoints** so no backend service is reachable
from the public internet, **Azure AD** for authentication (tokens via MSAL, RBAC enforced server-side
— see Course 5's dedicated RBAC/MSAL chapter for the pattern this project plausibly reused), and
**Azure Monitor / Application Insights** for observability and audit-grade tracing. This is a
sibling/adjacent system to Course 1's chatbot, not a replacement for it, and the two plausibly share
platform infrastructure (the same Azure OpenAI resource, the same VNet, the same observability
stack) even though they serve very different users and very different risk profiles.

> As with every course in this curriculum apart from Course 5 (which was rebuilt from real source
> code), **this course has no source repository behind it.** Everything below is a plausible,
> technically detailed reconstruction built from the resume bullet, standard AML/transaction-monitoring
> industry practice, and the same Azure RAG patterns established in Course 1. Any quantified
> metric is explicitly marked **illustrative** — treat it as a defensible placeholder to replace with
> your own real numbers, not a verified fact.

## Architecture Diagram

```mermaid
flowchart LR
    subgraph TM["Transaction Monitoring System (upstream, not built by this project)"]
        ALERT[Alert Triggered<br/>rule-based / ML-assisted]
    end

    subgraph DataLayer["Data-Gathering Layer"]
        KYC[(KYC Profile Store<br/>structured customer data)]
        TXN[(Core Banking / Transaction History<br/>structured, time-series)]
        CASES[(Prior Case Notes Store<br/>unstructured text)]
    end

    subgraph RAG["RAG / Retrieval Pipeline"]
        TOOL[Structured Query Tool<br/>KYC + transaction lookups<br/>function-calling, not embeddings]
        SEARCH[Azure AI Search<br/>semantic/hybrid retrieval<br/>over prior case notes]
    end

    subgraph GenLayer["Narrative Generation"]
        AOAI[Azure OpenAI<br/>structured-output drafting<br/>six-section template]
        CITE[Citation / Grounding Check<br/>every claim traces to a source]
    end

    subgraph Review["Human-in-the-Loop Review"]
        QUEUE[Compliance Officer<br/>Review Queue]
        SIGNOFF{Approve /<br/>Edit / Reject}
    end

    subgraph Record["Case Record"]
        FINAL[(Finalized Case Record<br/>+ full audit trail)]
        SAR[SAR Filing System<br/>only if escalated]
    end

    ALERT --> TOOL
    ALERT --> SEARCH
    KYC --> TOOL
    TXN --> TOOL
    CASES --> SEARCH
    TOOL --> AOAI
    SEARCH --> AOAI
    AOAI --> CITE
    CITE --> QUEUE
    QUEUE --> SIGNOFF
    SIGNOFF -->|Approved| FINAL
    SIGNOFF -->|Rejected| AOAI
    FINAL -->|if warranted| SAR
```

Plain-text version, if diagram rendering isn't available:

```
Transaction Monitoring System (upstream, rule-based/ML-assisted) triggers an Alert
  -> Data-Gathering Layer
       |-- KYC Profile Store (structured customer data)
       |-- Core Banking / Transaction History (structured, time-series, often high volume)
       `-- Prior Case Notes Store (unstructured text -- investigator write-ups from past cases)
  -> RAG / Retrieval Pipeline
       |-- Structured Query Tool: precise, auditable function-calling lookups against KYC and
       |   transaction data -- NOT embedded into a vector index (Chapter 2 explains why)
       `-- Azure AI Search: semantic/hybrid retrieval over prior case notes -- the one source
           that genuinely benefits from fuzzy semantic search (Chapter 2)
  -> Narrative Generation (Azure OpenAI)
       |-- Structured-output drafting against the six-section case-narrative template (Chapter 3)
       `-- Citation/grounding check -- every factual claim must cite a KYC field, transaction ID,
           or prior case ID (Chapter 3)
  -> Human-in-the-Loop Review (Chapter 4)
       |-- Draft lands in a Compliance Officer's review queue with per-section confidence indicators
       |-- Officer reviews, edits, and Approves or Rejects (Rejected -> regenerate)
       `-- ONLY an Approved narrative becomes part of the official case record -- this step is a
           hard requirement, not optional, see the load-bearing statement at the top of this file
  -> Finalized Case Record (with full audit trail: what was AI-drafted, what was human-edited, by whom)
  -> (if the officer's recommendation warrants it) SAR Filing System
```

Two details worth internalizing from the diagram before reading the chapters: **the structured query
tool and Azure AI Search are two different retrieval mechanisms feeding the same prompt**, not one
unified "RAG system" (Chapter 2 is the whole argument for why), and **the human-in-the-loop queue is
not a UI nicety bolted onto the end — it's the control that makes the rest of the architecture
defensible to a regulator** (Chapter 4).

## STAR Summary (practice this out loud, under 90 seconds)

> **Illustrative — replace with your real numbers before the interview.** The structure and reasoning
> are sound; the specific metrics (documentation time reduction, alert-to-closure turnaround) should be
> swapped for what you actually measured or a defensible estimate you're comfortable defending under
> follow-up questions.

**Situation.** At Capco, a global banking client's (plausibly HSBC's — say so honestly if you're not
certain) AML investigation team was spending a large share of each investigator's day manually
assembling case narratives for transaction-monitoring alerts — pulling KYC data from one system,
transaction history from core banking, and prior case notes from a case-management tool, then writing
up a structured narrative by hand for every alert, the overwhelming majority of which would turn out
to be false positives. That manual documentation burden was the actual bottleneck on alert-to-closure
turnaround, not the investigators' judgment.

**Task.** I was asked to architect a GenAI copilot that could auto-draft investigator-ready case
narratives — synthesizing KYC profiles, transaction history, and prior case notes into a structured,
citable document — while keeping a compliance officer firmly in the loop for sign-off, since a wrong
AML determination carries real regulatory and financial-crime consequences that make full automation
inappropriate for this domain.

**Action.** I designed a RAG pipeline that deliberately used **two different retrieval mechanisms** for
three source types: a structured, auditable query/function-calling tool for KYC and transaction data
(precise lookups, not fuzzy embedding search), and Azure AI Search's semantic/hybrid retrieval for
prior case notes, the one source that's genuinely unstructured text suited to that kind of search. I
used structured-output prompting against a six-section narrative template (Customer/KYC Overview,
Alert Trigger Summary, Transaction Pattern Analysis, Historical/Prior-Case Context, Red Flags
Identified, Recommendation) and required every factual claim in the draft to cite its source — a
specific KYC field, transaction ID, or prior case ID — as the primary anti-hallucination defense. I
built the human-in-the-loop review workflow as a real state machine (DRAFTED → UNDER_REVIEW →
APPROVED/REJECTED → FINALIZED, with a REGENERATE path on rejection) with a full audit trail of what
was AI-drafted versus human-edited, and designed a data-freshness check that flags a narrative for
regeneration if the customer's underlying data changed after the draft was generated but before
sign-off. All of it ran on the client's existing Azure production topology — Azure OpenAI, Azure AI
Search, Azure App Service, VNet/Private Endpoints, Azure AD — the same platform pattern as the AI
Chatbot Assistant project (Course 1).

**Result.** *(Illustrative)* The copilot reduced average per-alert documentation time by roughly 50%
and cut alert-to-closure turnaround by a comparable margin, while every finalized case record still
carried an explicit compliance-officer approval and a complete audit trail of the data it was grounded
in — the throughput gain came from drafting faster, not from reviewing less carefully.

## How This Course Is Organized

| File | Covers |
|---|---|
| `01-aml-transaction-monitoring-domain-fundamentals.md` | What triggers an alert, why false-positive rates are high, the six-section case-narrative structure, why this is a good-but-risky GenAI fit |
| `02-multi-source-rag-architecture-structured-vs-unstructured-retrieval.md` | The architecturally interesting part: why KYC and transaction data get a structured query tool instead of a vector index, why prior case notes are the one source that genuinely benefits from Azure AI Search semantic/hybrid retrieval, chunking strategy, and prompt assembly |
| `03-narrative-generation-and-structured-drafting.md` | Prompt engineering for a structured, investigator-ready document; structured-output/function-calling (cross-referencing Course 1 Chapter 1); the per-claim citation/grounding requirement as the primary anti-hallucination defense |
| `04-human-in-the-loop-compliance-sign-off.md` | The review/approval workflow in depth: why full automation is inappropriate here, the DRAFTED → UNDER_REVIEW → APPROVED/REJECTED → FINALIZED state machine, audit-trail requirements |
| `05-case-narrative-staleness-and-mid-review-data-changes.md` | What happens when new transactions arrive or a KYC profile changes while a narrative is drafted or sitting in review — today's gap, a proposed freshness check, and the audit-trail tie-in |
| `06-production-resilience-and-operational-engineering.md` | Realistic error-handling table, a batch-alert-burst concurrency caveat (cross-referencing Course 1 Chapter 7), domain-specific bug narratives, one named hardening gap |
| `07-data-compliance-and-regulatory-considerations.md` | Data residency, PII minimization in prompts, access control (cross-referencing Course 5's RBAC pattern), audit logging for regulatory examination, retention policy, vendor/model risk |
| `99-Interview-QA.md` | Behavioral, technical, system-design, and "what would you change" interview Q&A — leads with "why isn't this fully automated," featuring the staleness and data-compliance questions prominently |
| `notebooks/` | Five runnable Jupyter notebooks, one per major concept, fully offline |

Read in order — each chapter builds on the last, and the notebooks are meant to be run alongside the
chapter with the matching number.

---

Note: check your NDA before naming HSBC (or any specific client) by name in an actual interview —
"a global banking client" is both accurate to the resume bullet and always safe to say. See the root
README's confidentiality note.
