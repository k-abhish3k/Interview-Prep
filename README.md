# Interview Preparation Course — Abhishek Kumar

A tutorial-style deep-dive built from your resume. It covers 13 self-contained "courses," one per real
project from **Capco** and **Indegene**, plus a 14th folder — course 12, numbered before the two
newest additions — on production/development war stories.

Each course does three things:
1. Teaches the underlying concepts from first principles.
2. Ties those concepts back to what you actually built.
3. Ends with an interview-ready Q&A bank: behavioral questions, deep technical questions, and "what
   would you do differently."

## Client & Production Context (applies to every course below)

- **Capco (courses 1–6, 13)** — delivered for banking clients **HSBC** and **Bank of America**. All
  seven projects went to **production with a customer-facing interface**, not just a
  proof-of-concept. They ran on **Azure**, sitting behind Azure Front Door/Application Gateway (WAF +
  TLS termination), secured with **VNet integration / Private Endpoints** — a banking-grade
  requirement: no service should be reachable over the public internet without going through the
  gateway — with **Azure AD** for auth and **Azure Monitor / Application Insights** for monitoring.
  Courses 1–6 run on **Azure App Service**; course 13 is different — a self-hosted, open-weight model
  on **Azure ML / AKS** instead (see its course note for why). Each project's `00-README.md` and
  `99-Interview-QA.md` reflect this: daily production traffic, uptime expectations, and per-bank
  tenant isolation, meaning HSBC's data must never be visible inside Bank of America's environment,
  and vice versa.
- **Indegene (courses 7–11, 14)** — delivered for life-sciences clients **Eli Lilly**, **AstraZeneca**,
  and other top healthcare/pharma companies. All six projects went to **production with a
  customer-facing interface**, deployed on **AWS**. Courses 7–11 use **ECS (Fargate)** for the
  containerized application tier behind an **ALB**, **SageMaker** real-time endpoints for model
  inference, **Lambda + API Gateway** for event-driven/lightweight endpoints, **S3** for
  artifacts/documents, **CloudWatch** for observability, and **Secrets Manager** for credentials.
  Course 14 is different — a self-hosted, open-weight model on **AWS SageMaker batch/async
  inference** instead (see its course note for why). Pharma compliance considerations — GxP-style
  validation, content review/audit trails, PHI-adjacent data handling — are called out where relevant.
- **A confidentiality note before you use any of this in a real interview.** Consulting engagements
  are almost always covered by client-confidentiality clauses. Before naming HSBC, Bank of America,
  Eli Lilly, or AstraZeneca to an interviewer, check what your actual NDA/engagement letter allows —
  many candidates safely say "a top-3 global bank" or "a top-10 pharma company" instead. The content
  here uses the real names so *you* have full context while studying; swap in the safe phrasing for
  the interview itself if needed.

## How each course folder is organized

```
NN-Project-Name/
  00-README.md              Project overview: business context, your role, architecture, STAR summary
  01..0N-<topic>.md         Chapter-by-chapter concept deep dives (theory -> how it maps to the project)
  99-Interview-QA.md        Likely interview questions + model answers + follow-up probes
  notebooks/                Runnable Jupyter notebooks with hands-on, from-scratch examples
```

Read each course in this order: `00-README.md` → chapters `01..0N` → notebooks, run side by side with
the matching chapter → `99-Interview-QA.md` last, once the concepts are fresh.

## Course Map

> A recurring pair of chapters shows up across most courses: a **data-lifecycle/staleness-versioning
> chapter** and a **production-resilience-and-operational-engineering chapter**. This pattern was
> generalized from course 5's real-source-grounded rebuild (see its course note), then adapted to each
> project's own domain:
>
> - RAG knowledge staleness — course 1
> - Monitoring-baseline drift — course 4
> - Document revisions (the real one) — course 5
> - Multi-service version pinning — course 6
> - OCR reprocessing/dedup — course 7
> - Pinecone vector staleness — course 8
> - Approved-claims-library versioning — course 9
> - Detection-result dedup / model-version drift — course 10
> - Clinical protocol-amendment versioning — course 11
>
> Course 12 ties this whole pattern together across all of them. These are the newest, most
> technically dense chapters in those courses — read them last within each one, right before
> `99-Interview-QA.md`.
>
> Courses 2, 3, 13, and 14 are newer additions with their own equivalent chapters — see their own
> course notes: course 2's data-freshness-at-signoff chapter, course 3's model-version-pinning
> chapter, and courses 13/14's matched pair on base-model-vs-fine-tuning/taxonomy staleness.

| # | Folder | Company | Resume Project | Core Skills Covered |
|---|--------|---------|-----------------|----------------------|
| 1 | [01-Capco-AI-Chatbot-Assistant](01-Capco-AI-Chatbot-Assistant/00-README.md) | Capco | AI Chatbot Assistant | Python, Azure OpenAI, LangChain, LCEL, prompt engineering, RAG basics, Azure DevOps, `react-service` frontend microservice (production UI — not Chainlit/Streamlit), guardrails/scope & vague-query handling |
| 2 | [02-AML-Alert-Investigation-Copilot](02-AML-Alert-Investigation-Copilot/00-README.md) | Capco | AML Alert Investigation Copilot | RAG (Azure OpenAI + Azure AI Search), structured/function-calling vs. semantic retrieval, structured-output narrative drafting, per-claim citation grounding, human-in-the-loop compliance sign-off, AML/regulatory data compliance |
| 3 | [03-Claude-Research-Project](03-Claude-Research-Project/00-README.md) | Capco | Multi-Agent Credit Underwriting Assistant | Multi-agent orchestration (LangGraph supervisor pattern), Azure AI Foundry model catalog vs. Azure OpenAI Service, agentic/tool-use evaluation (builds on course 4's Ragas methodology), shared-context synthesis across agents, shadow/canary production rollout, new-vendor data-compliance & risk assessment, model-version pinning, agent failure isolation & runaway-loop protection |
| 4 | [04-Capco-Model-Risk-Monitoring](04-Capco-Model-Risk-Monitoring/00-README.md) | Capco | Model Risk Monitoring for AI Assistant | LLM evaluation, Ragas, hallucination/robustness/harmfulness metrics, Shapash, Lime, explainable AI |
| 5 | [05-Capco-Document-Uploader-Service](05-Capco-Document-Uploader-Service/00-README.md) | Capco | Document Uploader Service (rebuilt from real source — see course note) | REST APIs, FastAPI, SQLAlchemy ORM, RBAC with MSAL/OAuth, multi-department routing, document versioning, Azure App Service (native, no Docker), SQL |
| 6 | [06-Capco-CICD-Pipelines](06-Capco-CICD-Pipelines/00-README.md) | Capco | CI/CD Pipeline Implementation | Azure DevOps, YAML pipelines, Terraform, GitHub, release management |
| 7 | [07-Indegene-Superscript-Subscript-Detection](07-Indegene-Superscript-Subscript-Detection/00-README.md) | Indegene | Superscript/Subscript Detection | OCR, YOLO object detection, CNN classification, OpenCV, AWS Sagemaker |
| 8 | [08-Indegene-GenAI-Virtual-Liaison-Platform](08-Indegene-GenAI-Virtual-Liaison-Platform/00-README.md) | Indegene | GenAI Virtual Liaison Platform | RAG with LangChain + Pinecone, HybridSearch RAG, summarization, entity extraction, recommenders, Langgraph, Langserve |
| 9 | [09-Indegene-Claim-Extraction-Tagging](09-Indegene-Claim-Extraction-Tagging/00-README.md) | Indegene | Claim Extraction & Tagging | NLP classification, BERT/Transformers, AWS Sagemaker fine-tuning, Lambda, EventBridge, MLOps pipelines |
| 10 | [10-Indegene-Chart-Graph-Detection](10-Indegene-Chart-Graph-Detection/00-README.md) | Indegene | Chart/Graph Detection | AWS Ground Truth labeling, YOLOv5 fine-tuning, IOU/TPR metrics, Lambda + API Gateway deployment |
| 11 | [11-Indegene-GenAI-Regulatory-Platform](11-Indegene-GenAI-Regulatory-Platform/00-README.md) | Indegene | GenAI Regulatory Platform (ICF/PLPS/SOC) | Instruction-based generation, document structuring, clinical NLP, LLAMA 3, LLM finetuning, GraphDB/Cypher |
| 12 | [12-Toughest-Challenges-Production-and-Development](12-Toughest-Challenges-Production-and-Development/00-README.md) | Both | Cross-project war stories | Production scaling/reliability, security/compliance, observability, cost/quota management, data scarcity, LLM eval, legacy integration, cross-functional friction |
| 13 | [13-Regulatory-Policy-Intelligence-Platform](13-Regulatory-Policy-Intelligence-Platform/00-README.md) | Capco | Regulatory & Policy Document Intelligence Platform | Mixtral 8x7B architecture (GQA, sliding-window attention, sparse MoE — real, published), self-hosted open-weight vs. managed-API tradeoffs, Azure ML/AKS deployment, quantization, LoRA fine-tuning, model supply-chain/provenance verification, base-model vs. taxonomy staleness |
| 14 | [14-Systematic-Literature-Review-Copilot](14-Systematic-Literature-Review-Copilot/00-README.md) | Indegene | Systematic Literature Review (SLR) Copilot | DeepSeek architecture (Multi-head Latent Attention, fine-grained MoE, auxiliary-loss-free load balancing — real, published), self-hosted open-weight vs. managed-API tradeoffs, AWS Sagemaker batch/async inference, LoRA fine-tuning, model provenance verification, model-version vs. screening-criteria staleness |

## Full Skill Coverage Matrix

See [SKILLS-MATRIX.md](SKILLS-MATRIX.md) for exactly which resume skill is taught in which course —
every bullet from your "Strengths and Expertise" section is mapped to at least one chapter.

## Suggested Study Plan

- **Week 1**: Courses 1, 2, 3, 4, 5, 6 — the Capco / GenAI backend + DevOps track.
- **Week 2**: Courses 7, 10 — the Computer Vision track.
- **Week 3**: Courses 8, 9, 11 — the Indegene GenAI/NLP track.
- **Week 4**: Courses 13, 14 — the open-weight model architecture + self-hosting track. Read these two
  together; they're designed as a matched pair — same chapter shape, different cloud, different model,
  same "what's genuinely known vs. undisclosed" discipline that course 3's Chapter 8 introduces.
- **Final pass**: Re-read every `99-Interview-QA.md`. Then do a mock interview covering one project
  from each company, chosen at random.

## General Interview Strategy

- Every project doc's **STAR summary** (Situation/Task/Action/Result) is your answer skeleton for
  "tell me about a project where you..." questions. Practice saying it out loud in under 90 seconds.
- Have **one number** ready per project — accuracy uplift, TPR, latency, cost saved, time saved.
  Interviewers remember quantified results.
- For every project, be ready for the follow-up **"what would you change if you rebuilt this
  today?"** Each `99-Interview-QA.md` includes a suggested answer.
- Course 12 is your answer bank for the near-universal "tell me about your most challenging technical
  problem" and "tell me about a production incident" questions. Read it last, right before the
  interview.
