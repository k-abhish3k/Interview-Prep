# Skills Coverage Matrix

Every item from the resume's "Strengths and Expertise" section, mapped to the course(s) that teach it.
Use this as a checklist — if an interviewer asks about any resume bullet, you'll know exactly which
folder to review.

## Generative AI

| Skill | Primary Course(s) |
|---|---|
| LLM fundamentals, Prompt engineering | 01-Capco-AI-Chatbot-Assistant |
| LangChain, LCEL, LangServe | 01-Capco-AI-Chatbot-Assistant, 08-Indegene-GenAI-Virtual-Liaison-Platform |
| LangGraph | 08-Indegene-GenAI-Virtual-Liaison-Platform, 03-Claude-Research-Project (multi-agent Supervisor pattern) |
| LLAMA 3, LLM Finetuning | 11-Indegene-GenAI-Regulatory-Platform |
| Chatbot design | 01-Capco-AI-Chatbot-Assistant |
| Explainable AI, Shapash, LIME | 04-Capco-Model-Risk-Monitoring |
| Chainlit, Streamlit | 01-Capco-AI-Chatbot-Assistant (rapid POC/prototyping tool — the real production UI is `react-service`, a dedicated React frontend microservice; see chapter 04's correction) |
| Guardrails: scope, vague-query, and off-topic handling | 01-Capco-AI-Chatbot-Assistant |
| Ragas (RAG evaluation) | 04-Capco-Model-Risk-Monitoring, 03-Claude-Research-Project (agentic/multi-agent task-completion eval) |
| HybridSearch RAG | 08-Indegene-GenAI-Virtual-Liaison-Platform |
| Cypher query language, GraphDB | 11-Indegene-GenAI-Regulatory-Platform |
| Structured-output/function-calling as a retrieval design choice (not just formatting) | 02-AML-Alert-Investigation-Copilot |
| Per-claim citation grounding as an automated anti-hallucination check | 02-AML-Alert-Investigation-Copilot |
| Azure AI Foundry model catalog (Claude, Llama, Mistral) vs. Azure OpenAI Service | 03-Claude-Research-Project |
| Multi-agent orchestration, agentic evaluation, shadow-mode production rollout | 03-Claude-Research-Project |
| Claude architecture vs. GPT-4 (what's published vs. undisclosed for both) | 03-Claude-Research-Project (chapter 08) |
| Mixtral 8x7B architecture: GQA, sliding-window attention, sparse MoE (top-2-of-8 routing) | 13-Regulatory-Policy-Intelligence-Platform |
| DeepSeek architecture: Multi-head Latent Attention, fine-grained MoE, auxiliary-loss-free load balancing | 14-Systematic-Literature-Review-Copilot |
| Self-hosted open-weight model vs. managed API tradeoffs (cost, fine-tuning control, weight-level transparency) | 13-Regulatory-Policy-Intelligence-Platform, 14-Systematic-Literature-Review-Copilot |
| Quantization cost/latency/quality tradeoffs | 13-Regulatory-Policy-Intelligence-Platform |
| Model supply-chain / weight provenance verification | 13-Regulatory-Policy-Intelligence-Platform, 14-Systematic-Literature-Review-Copilot |

## DevOps

| Skill | Primary Course(s) |
|---|---|
| Azure DevOps, CI/CD Pipelines | 06-Capco-CICD-Pipelines |
| Terraform | 06-Capco-CICD-Pipelines |
| WebApp, Azure Functions | 05-Capco-Document-Uploader-Service |
| Sagemaker Studio | 07, 09, 10 (Indegene CV/NLP courses) |
| REST APIs, FastAPI, SQLAlchemy (ORM) | 05-Capco-Document-Uploader-Service |
| Docker | 06-Capco-CICD-Pipelines (05's real deployment is native App Service, no Docker — see course 05 note) |
| TensorFlow Serving | 10-Indegene-Chart-Graph-Detection |
| Power BI, Tableau | 04-Capco-Model-Risk-Monitoring (monitoring dashboards) |
| Azure ML, SQL | 05-Capco-Document-Uploader-Service |
| GitHub | 06-Capco-CICD-Pipelines |

## Cloud Services

| Skill | Primary Course(s) |
|---|---|
| Azure AI Search | 08-Indegene-GenAI-Virtual-Liaison-Platform (as Azure equivalent), 01 |
| Azure WebApp, Azure Functions | 05-Capco-Document-Uploader-Service |
| Azure Key Vault | 05-Capco-Document-Uploader-Service (real config is App Service Application Settings; Key Vault is a proposed hardening step, not confirmed as implemented) |
| Azure Cognitive Services, Azure OpenAI | 01-Capco-AI-Chatbot-Assistant |
| Microsoft Graph API | 05-Capco-Document-Uploader-Service |
| Azure ML managed online endpoints, AKS (self-hosted model serving) | 13-Regulatory-Policy-Intelligence-Platform |
| AWS Secrets Manager | 09-Indegene-Claim-Extraction-Tagging |
| AWS Sagemaker | 07, 09, 10, 14 (14 uses Batch Transform/async inference, not just real-time) |
| AWS Bedrock | 11-Indegene-GenAI-Regulatory-Platform (alt LLM backend) |
| S3, Lambda, API Gateway | 09-Indegene-Claim-Extraction-Tagging, 10-Indegene-Chart-Graph-Detection |
| AWS Textract | 07-Indegene-Superscript-Subscript-Detection |
| AWS Rekognition | 10-Indegene-Chart-Graph-Detection |
| AWS Comprehend | 09-Indegene-Claim-Extraction-Tagging |
| AWS CloudWatch, EventBridge | 09-Indegene-Claim-Extraction-Tagging |
| AWS Step Functions / Batch (large-scale pipeline orchestration) | 14-Systematic-Literature-Review-Copilot |

## Deep Learning & Computer Vision

| Skill | Primary Course(s) |
|---|---|
| Scikit-learn, TensorFlow, Keras, PyTorch (torch) | 07, 10 |
| OpenCV | 07-Indegene-Superscript-Subscript-Detection |
| GANs (incl. SRGAN) | 10-Indegene-Chart-Graph-Detection (bonus chapter), notebooks |
| CNNs, Object Detection (YOLO/YOLOv5), Semantic Segmentation | 07, 10 |
| Transformers, GPT-4 Vision | 07-Indegene-Superscript-Subscript-Detection |

## NLP

| Skill | Primary Course(s) |
|---|---|
| NLTK, spaCy, Transformers | 09-Indegene-Claim-Extraction-Tagging |
| Conversational AI | 01-Capco-AI-Chatbot-Assistant, 08 |
| Seq2Seq, Transfer Learning | 09-Indegene-Claim-Extraction-Tagging, 11 |
| Word2Vec, BERT | 09-Indegene-Claim-Extraction-Tagging |

## Data Analysis & Visualization

| Skill | Primary Course(s) |
|---|---|
| Pandas, NumPy, Matplotlib | woven through all notebooks |
| SQL | 05-Capco-Document-Uploader-Service |

## Personal projects (folded in as bonus chapters)

| Project | Folded into |
|---|---|
| Text-to-Math Solver (LangChain agent + tools, Groq/Gemma2) | 01-Capco-AI-Chatbot-Assistant (agents chapter) |
| Real-time Super Resolution (SRGAN) | 10-Indegene-Chart-Graph-Detection (GAN bonus chapter) |
| ML-based Crop Production | 09-Indegene-Claim-Extraction-Tagging (classic ML bonus chapter) |

## Production Cloud Operations (client-facing deployments)

All 11 numbered project courses went to production with a customer/loan-officer-facing interface — see
the "Client & Production Context" section in the root `README.md` for the HSBC/Bank of America (Capco)
and Eli Lilly/AstraZeneca (Indegene) client mapping.

| Skill | Primary Course(s) |
|---|---|
| Azure App Service production hosting (deployment slots, autoscale) | 01, 04, 05, 06 |
| Azure Front Door / Application Gateway, WAF, VNet/Private Endpoints | 05, 06, 12 |
| Azure AD auth, Azure Monitor / Application Insights | 01, 04, 06, 12 |
| AWS ECS (Fargate) + ALB as the production application tier | 07, 08, 09, 10, 11, 12 |
| AWS Sagemaker real-time inference endpoints | 07, 09, 10 |
| AWS Lambda + API Gateway (event-driven/lightweight endpoints) | 09, 10, 12 |
| CloudWatch / Secrets Manager (AWS side), production observability | 09, 10, 11, 12 |
| Multi-tenant client data isolation, banking/pharma compliance | 12 |

## Authentication & Authorization

| Skill | Primary Course(s) |
|---|---|
| OAuth 2.0 / OpenID Connect fundamentals | 05-Capco-Document-Uploader-Service, 01 (light touch) |
| MSAL (Microsoft Authentication Library), Azure AD app registrations | 05-Capco-Document-Uploader-Service, 01 (light touch) |
| RBAC (Role-Based Access Control) design | 05-Capco-Document-Uploader-Service |

## Regulated-Domain Compliance & Governance

| Skill | Primary Course(s) |
|---|---|
| AML/transaction-monitoring domain knowledge (alerts, KYC, SAR-adjacent decisions) | 02-AML-Alert-Investigation-Copilot |
| Human-in-the-loop sign-off workflow design (state machine, audit trail) | 02-AML-Alert-Investigation-Copilot, 11-Indegene-GenAI-Regulatory-Platform |
| Data-freshness/staleness checks at approval time | 02-AML-Alert-Investigation-Copilot |
| Data compliance & vendor risk assessment for a new LLM provider | 03-Claude-Research-Project |
| Regulatory audit-trail design (what was AI-drafted vs. human-edited) | 02-AML-Alert-Investigation-Copilot, 11-Indegene-GenAI-Regulatory-Platform |
