# 03 — Chatbot Architecture on Azure OpenAI

## Why this chapter matters

"Azure Cloud" and "Azure OpenAI" are both named explicitly in the resume bullet. This is the chapter
that turns those two words into a defensible system design — the kind of thing you'd sketch on a
whiteboard if asked "walk me through how you'd architect this chatbot in production." This chatbot
did, in fact, run in production for **HSBC** with a customer-facing interface (see `00-README.md`'s
"Client & Production Deployment" section and the "Running This in Production for a Bank" section
below for the confirmed production topology). The fine-grained implementation detail in the rest of
this chapter — exact retrieval/session-store choices, specific tradeoff numbers — should still be
treated as **"a typical/recommended approach for this class of project"**, not a verified
line-by-line description of Capco's actual internal system.

## Azure OpenAI Service: Deployments, Models, Rate Limits

Azure OpenAI Service gives an organization access to OpenAI's models (GPT-3.5, GPT-4, GPT-4o,
embedding models, etc.) hosted inside Azure, inside the client's own subscription/tenant boundary —
the core reason a financial-services or energy client would choose it over calling the public OpenAI
API: **data residency and compliance**. Requests and the data inside them stay within the
organization's Azure environment and are subject to Azure's enterprise agreements, rather than
transiting to a third-party's public API surface.

Key operational concepts:

- **Deployments**: you don't call "gpt-4" directly — you create a *deployment* of a specific model
  version inside your Azure OpenAI resource, give it a deployment name, and your application code
  targets that name. This indirection lets you upgrade the underlying model version without changing
  application code, and lets you run multiple deployments (e.g., a cheaper model for classification,
  a stronger model for final answer generation) side by side.
- **Rate limits (TPM/RPM)**: Azure OpenAI enforces tokens-per-minute and requests-per-minute quotas
  per deployment. A chatbot with unpredictable, bursty traffic (e.g., a spike when a new policy
  document ships and everyone asks about it at once) needs to handle `429` throttling responses
  gracefully — exponential backoff with retry, and ideally a request queue rather than surfacing the
  error straight to the user.
- **Content filtering**: Azure OpenAI applies a built-in content filter to both prompts and
  completions (categories like violence, hate, sexual content, self-harm), which can block or flag a
  response independently of your own application logic. For a financial-services chatbot this is
  mostly a non-issue in the mainline flow, but it matters for error handling — your backend needs to
  catch a content-filter rejection and respond to the user with something more graceful than a raw
  API error, and it's a detail worth mentioning if asked about production robustness.

## Conversation State / Session Management

As covered in Chapter 1, the model itself is stateless — every "memory" of the conversation is
something your backend re-sends on every call. In production this means:

- A **session store** keyed by user/conversation ID (Redis, Azure Cosmos DB, or even a simple table
  in Azure SQL for a lower-traffic internal tool) holding the running message history.
- A **token-budget strategy** for what to include as that history grows: send the full history until
  it approaches the model's context window, then switch to a sliding window (keep only the last N
  turns) or a summarization strategy (periodically compress older turns into a short summary message
  that replaces them) — see the hands-on truncation example in
  `notebooks/03_simple_chatbot_with_memory.ipynb`.
- **Statelessness at the compute layer**: the App Service/Function instances handling requests should
  themselves be stateless, pulling conversation state from the session store on each request — this
  is what lets you scale horizontally behind a load balancer without sticky sessions (see also the
  system-design question on scaling to 10k concurrent users in `99-Interview-QA.md`).

## Retrieval-Augmented Generation Basics (Azure AI Search)

A chatbot answering "how long do refunds take" correctly needs the actual policy text in its prompt
— that's RAG (retrieval-augmented generation), and on Azure the natural retrieval component is
**Azure AI Search** (formerly Azure Cognitive Search), which supports:

- **Vector search**: documents are chunked and embedded (via an Azure OpenAI embedding deployment,
  e.g. `text-embedding-ada-002` / `text-embedding-3-small`) into a vector index; a user's question is
  embedded the same way and the index returns the semantically nearest chunks.
- **Hybrid search**: combining vector similarity with traditional keyword (BM25) search, which
  typically outperforms either alone — vector search catches semantic matches keyword search misses
  ("refund" vs "money back"), keyword search catches exact terms vector search can blur (policy
  numbers, product codes).
- **Semantic ranking**: an optional re-ranking layer that reorders the initial result set by
  relevance to the query using a cross-encoder-style model.

The RAG flow, end to end: user question → embed question → query Azure AI Search index → retrieve
top-k chunks → insert chunks into the prompt as `{context}` (as in the `ChatPromptTemplate` from
Chapter 2) → Azure OpenAI generates the grounded answer → (ideally) return the source
document/section alongside the answer for the user to verify. This is also the primary defense
against hallucination discussed in Chapter 1 — the model is instructed to answer *from the provided
context* rather than from parametric memory, and to say "I don't know" when the retrieved context
doesn't cover the question.

> Note: a smaller-scope internal chatbot at this stage of a project might not have full RAG — it
> could be a scoped-instructions bot backed by a fixed FAQ, before scaling to full retrieval. Frame
> your answer around what's plausible for the resume bullet's apparent scope, and be honest in an
> interview about which parts you're confident on vs. inferring.

## Latency and Cost Tradeoffs

Every architectural choice here trades latency and cost against quality:

| Lever | Faster / Cheaper | Slower / More Expensive | Better Quality |
|---|---|---|---|
| Model choice | GPT-3.5-class deployment | — | GPT-4-class deployment |
| Retrieval depth | Fewer chunks (top-3) | More chunks (top-10+) | More context, but risk of window overflow and noise |
| Streaming | — | — | Streaming doesn't reduce total latency but drastically improves *perceived* latency (Chapter 2) |
| Caching | Cache embeddings / common Q&A pairs | Full RAG on every call | — |
| Prompt length | Short, tight system prompt | Long few-shot examples | Longer prompts often mean better-controlled output at the cost of more tokens billed |

A practical rule of thumb worth stating in an interview: measure **time-to-first-token** separately
from **total generation time** — streaming makes the former the number users actually feel, and
optimizing retrieval latency (the AI Search call) often matters more to perceived responsiveness than
shaving tokens off the LLM call itself.

## Content Filtering & Safety

Beyond Azure OpenAI's built-in content filter, a chatbot for a regulated client typically layers on:

- **Scope guarding** via the system prompt (Chapter 1) — refuse to answer outside the intended
  domain (e.g., no general financial advice, no requests unrelated to the client's documented
  policies).
- **PII handling** — depending on the use case, redacting or avoiding echoing sensitive personal data
  back into logs (Application Insights) or into the LLM prompt unnecessarily.
- **Audit logging** — every question/answer pair logged with a timestamp, user ID, and (if RAG) the
  source documents used, both for compliance and for later evaluation (see Course 04,
  Model Risk Monitoring, for the evaluation/monitoring layer that would typically sit on top of this
  chatbot in a mature deployment).

## Running This in Production for a Bank

Everything above is what makes the chatbot *answer well*. This section is what makes it *safe to run
in production, customer-facing, at a bank* — this project (built at Capco for **HSBC**, with a
comparable pattern later for Bank of America) actually shipped to production with real daily
customer/employee traffic, not just a pilot, so this isn't hypothetical. Five things change:

- **Private networking, not public endpoints.** In a pilot/POC, it's common for the App
  Service/Function and its dependencies to sit on public endpoints protected only by keys. In
  production for a bank, the App Service is deployed **inside a VNet**, and it reaches Azure OpenAI,
  Azure AI Search, and Key Vault through **Private Endpoints** — meaning those services have no
  public IP reachable from the internet at all, only from inside the private network boundary. This
  is a hard requirement, not a nice-to-have, for a banking client.
- **WAF and DDoS protection at the edge.** Nothing hits the App Service directly. Traffic first
  passes through **Azure Front Door or Application Gateway**, which terminates TLS and applies a
  **Web Application Firewall** — blocking common attack patterns (SQL injection, XSS, known bad
  request signatures) and absorbing volumetric abuse before it ever reaches application compute. This
  is also the layer where you'd rate-limit or geo-restrict traffic if the client required it.
- **Azure AD-based auth instead of open access.** A pilot chatbot might be reachable by anyone with
  the URL. The production version authenticates every user through **Azure AD**, with the client/backend
  acquiring tokens via **MSAL** (Microsoft's OAuth 2.0/OIDC library) rather than hand-rolling token
  handling, so access maps to real bank identities (employees, or authenticated customers depending on
  the surface). Authentication only answers "who is this?" — **RBAC** (role-based access control) on
  top of the validated token's claims is what answers "what are they allowed to do?" (e.g., a
  `ChatUser` role vs. a `ChatAdmin` role with access to conversation-review tooling). There's no
  anonymous path into a system that talks to Azure OpenAI on the bank's behalf. See Course 05's
  dedicated chapter for the full OAuth/MSAL/RBAC implementation pattern shared across these backend
  services.
- **Deployment slots for zero-downtime releases.** Azure App Service supports multiple **deployment
  slots** (e.g., `staging` and `production`) backed by the same App Service Plan. A new build deploys
  and warms up in the staging slot — including running smoke tests against it — and only then is
  **slot-swapped** into production, which is close to instantaneous and avoids the downtime window of
  a straight in-place redeploy. If something's wrong post-swap, swapping back is just as fast. This is
  the deployment mechanic that the Azure DevOps pipeline in Course 06 targets; Course 06 covers the
  pipeline stages in depth, this chapter is just the "why slots" piece of that story.
- **Application Insights tracing per conversation, for audit/compliance.** Beyond generic uptime
  monitoring, every conversation gets end-to-end **Application Insights** tracing: request ID,
  authenticated user identity, retrieved document sources (if RAG), model/deployment version used,
  token counts, latency breakdown per stage (retrieval vs. generation), and the content-filter
  outcome. For a bank, this isn't just for debugging — it's the audit trail that lets compliance
  reconstruct "who asked what, what the system answered, and what it was grounded in" for any given
  conversation after the fact.

Taken together, these are the answer to "what's different about a demo chatbot versus one you'd
actually put in front of HSBC's customers" — the RAG/prompt-engineering substance doesn't change, but
every request now flows through a WAF, an identity check, and a private network boundary, and every
conversation leaves an audit trail, before it ever reaches Azure OpenAI.

## Tying It Back

This is the chapter to lean on for "design me the chatbot" style system-design questions: Azure
OpenAI deployment(s) behind a stateless Python API layer, a session store for conversation memory, a
retrieval layer via Azure AI Search for grounding, streaming responses back to the UI, and content
filtering plus audit logging as the safety net appropriate for a financial-services/energy client. In
production for a bank, add the edge/identity/network layer above: Front Door/App Gateway (WAF) ->
Azure AD -> App Service in a VNet with Private Endpoints -> Azure OpenAI, with Application Insights
tracing every conversation. Azure DevOps (Course 06 goes deeper on this) is what ships changes to any
layer of this stack safely, via deployment slots for zero-downtime releases.
