# Interview Preparation Course — Abhishek Kumar

A tutorial-style deep-dive built from your resume: 9 self-contained "courses," one per real project
from **Capco** and **Indegene**, plus a 10th folder on production/development war stories. Each course
teaches the underlying concepts from first principles, then ties them back to what you actually built,
and ends with an interview-ready Q&A bank (behavioral + deep technical + "what would you do differently").

## Client & Production Context (applies to every course below)

- **Capco (courses 1–4)**: delivered for banking clients **HSBC** and **Bank of America**. All four
  projects went to **production with a customer-facing interface** — not a proof-of-concept — deployed
  on **Azure App Service**, fronted by Azure Front Door/Application Gateway (WAF + TLS termination),
  secured with **VNet integration / Private Endpoints** (a banking-grade requirement — no service should
  be reachable over the public internet without going through the gateway), **Azure AD** for auth, and
  monitored with **Azure Monitor / Application Insights**. Each project's `00-README.md` and
  `99-Interview-QA.md` now reflect this: daily production traffic, uptime expectations, and per-bank
  tenant isolation (HSBC's data must never be visible to Bank of America's environment, and vice versa).
- **Indegene (courses 5–9)**: delivered for life-sciences clients **Eli Lilly**, **AstraZeneca**, and
  other top healthcare/pharma companies. All five projects went to **production with a customer-facing
  interface**, deployed on **AWS** using **ECS (Fargate)** for the containerized application tier behind
  an **ALB**, **Sagemaker** real-time endpoints for model inference, **Lambda + API Gateway** for
  event-driven/lightweight endpoints, **S3** for artifacts/documents, **CloudWatch** for observability,
  and **Secrets Manager** for credentials. Pharma compliance considerations (GxP-style validation,
  content review/audit trails, PHI-adjacent data handling) are called out where relevant.
- **A confidentiality note before you use any of this in a real interview**: consulting engagements are
  almost always covered by client-confidentiality clauses. Before naming HSBC, Bank of America, Eli
  Lilly, or AstraZeneca by name to an interviewer, check what your actual NDA/engagement letter allows —
  many candidates safely say "a top-3 global bank" or "a top-10 pharma company" instead. The content here
  uses the real names so *you* have full context while studying; swap in the safe phrasing for the
  interview itself if needed.

## How each course folder is organized

```
NN-Project-Name/
  00-README.md              Project overview: business context, your role, architecture, STAR summary
  01..0N-<topic>.md         Chapter-by-chapter concept deep dives (theory -> how it maps to the project)
  99-Interview-QA.md        Likely interview questions + model answers + follow-up probes
  notebooks/                Runnable Jupyter notebooks with hands-on, from-scratch examples
```

Read each course in order: `00-README.md` -> chapters `01..0N` -> notebooks (run them side by side
with the corresponding chapter) -> `99-Interview-QA.md` last, once the concepts are fresh.

## Course Map

| # | Folder | Company | Resume Project | Core Skills Covered |
|---|--------|---------|-----------------|----------------------|
| 1 | [01-Capco-AI-Chatbot-Assistant](01-Capco-AI-Chatbot-Assistant/00-README.md) | Capco | AI Chatbot Assistant | Python, Azure OpenAI, LangChain, LCEL, prompt engineering, RAG basics, Azure DevOps |
| 2 | [02-Capco-Model-Risk-Monitoring](02-Capco-Model-Risk-Monitoring/00-README.md) | Capco | Model Risk Monitoring for AI Assistant | LLM evaluation, Ragas, hallucination/robustness/harmfulness metrics, Shapash, Lime, explainable AI |
| 3 | [03-Capco-Document-Uploader-Service](03-Capco-Document-Uploader-Service/00-README.md) | Capco | Document Uploader Service (rebuilt from real source — see course note) | REST APIs, FastAPI, SQLAlchemy ORM, RBAC with MSAL/OAuth, multi-department routing, document versioning, Azure App Service (native, no Docker), SQL |
| 4 | [04-Capco-CICD-Pipelines](04-Capco-CICD-Pipelines/00-README.md) | Capco | CI/CD Pipeline Implementation | Azure DevOps, YAML pipelines, Terraform, GitHub, release management |
| 5 | [05-Indegene-Superscript-Subscript-Detection](05-Indegene-Superscript-Subscript-Detection/00-README.md) | Indegene | Superscript/Subscript Detection | OCR, YOLO object detection, CNN classification, OpenCV, AWS Sagemaker |
| 6 | [06-Indegene-GenAI-Virtual-Liaison-Platform](06-Indegene-GenAI-Virtual-Liaison-Platform/00-README.md) | Indegene | GenAI Virtual Liaison Platform | RAG with LangChain + Pinecone, HybridSearch RAG, summarization, entity extraction, recommenders, Langgraph, Langserve |
| 7 | [07-Indegene-Claim-Extraction-Tagging](07-Indegene-Claim-Extraction-Tagging/00-README.md) | Indegene | Claim Extraction & Tagging | NLP classification, BERT/Transformers, AWS Sagemaker fine-tuning, Lambda, EventBridge, MLOps pipelines |
| 8 | [08-Indegene-Chart-Graph-Detection](08-Indegene-Chart-Graph-Detection/00-README.md) | Indegene | Chart/Graph Detection | AWS Ground Truth labeling, YOLOv5 fine-tuning, IOU/TPR metrics, Lambda + API Gateway deployment |
| 9 | [09-Indegene-GenAI-Regulatory-Platform](09-Indegene-GenAI-Regulatory-Platform/00-README.md) | Indegene | GenAI Regulatory Platform (ICF/PLPS/SOC) | Instruction-based generation, document structuring, clinical NLP, LLAMA 3, LLM finetuning, GraphDB/Cypher |
| 10 | [10-Toughest-Challenges-Production-and-Development](10-Toughest-Challenges-Production-and-Development/00-README.md) | Both | Cross-project war stories | Production scaling/reliability, security/compliance, observability, cost/quota management, data scarcity, LLM eval, legacy integration, cross-functional friction |

## Full Skill Coverage Matrix

See [SKILLS-MATRIX.md](SKILLS-MATRIX.md) for exactly which resume skill is taught in which course —
every bullet from your "Strengths and Expertise" section is mapped to at least one chapter.

## Suggested Study Plan

- **Week 1**: Courses 1, 2, 3, 4 (Capco / GenAI backend + DevOps track)
- **Week 2**: Courses 5, 8 (Computer Vision track)
- **Week 3**: Courses 6, 7, 9 (Indegene GenAI/NLP track)
- **Final pass**: Re-read every `99-Interview-QA.md`, then do a mock interview covering one project
  from each company chosen at random.

## General Interview Strategy

- Every project doc's **STAR summary** (Situation/Task/Action/Result) is your answer skeleton for
  "tell me about a project where you..." questions. Practice saying it out loud in under 90 seconds.
- Have **one number** ready per project (accuracy uplift, TPR, latency, cost saved, time saved) —
  interviewers remember quantified results.
- For every project, be ready for the follow-up **"what would you change if you rebuilt this today?"**
  — each `99-Interview-QA.md` includes a suggested answer.
- Course 10 is your answer bank for the near-universal "tell me about your most challenging technical
  problem" and "tell me about a production incident" questions — read it last, right before the interview.
