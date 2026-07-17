# 99 — Interview Q&A: Capco AI Chatbot Assistant

Read this last, after the chapters and notebooks are fresh. Practice the behavioral answer out loud
first — it's your opener for almost any "tell me about a project" prompt.

---

### 1. (Behavioral) Tell me about this project.

Use the STAR summary from `00-README.md` verbatim, practiced to under 90 seconds:

> At Capco, a financial-services client's support and ops teams were losing time manually searching
> internal documentation to answer routine questions. I was asked to build an intelligent chatbot
> that could understand natural-language questions and return accurate, grounded answers, entirely
> within the client's Azure environment. I built the backend in Python using Azure OpenAI as the LLM
> provider, composed the pipeline with LangChain/LCEL, added prompt engineering and conversation
> memory to keep answers accurate and context-aware, shipped a Streamlit/Chainlit UI for fast
> internal testing, and set up the Azure DevOps CI/CD pipeline so changes deployed safely through
> dev/test/prod. *(Illustrative)* The pilot cut query resolution time by roughly 35% and deflected
> about 40% of routine support tickets.

Remember to swap the italicized result for your real numbers before using this live.

---

### 2. Walk me through what happens, end to end, when a user sends a message to the chatbot.

Message arrives at the Python API layer (Azure App Service/Functions) → the backend pulls the
user's conversation history from the session store → (if RAG-enabled) the question is embedded and
used to query Azure AI Search for relevant document chunks → those chunks plus the history are
inserted into a `ChatPromptTemplate` → the LCEL chain (`prompt | llm | parser`) invokes Azure OpenAI
→ the response streams token-by-token back through the API to the UI, where it renders incrementally
→ the full exchange is appended to the session store and logged for audit/monitoring. See
`00-README.md`'s architecture diagram and Chapter 3 for the full picture.

---

### 3. How would you reduce hallucination in this chatbot?

Layered defenses, from cheapest to most involved: (1) a tight system prompt that explicitly instructs
the model to answer only from provided context and to say "I don't know" otherwise (Chapter 1); (2)
low temperature (0–0.3) so answers are consistent rather than creative; (3) grounding via RAG —
retrieving the actual relevant policy text via Azure AI Search and inserting it into the prompt, so
the model has the correct facts in context rather than relying on parametric memory (Chapter 3); (4)
structured output with a required "source" field, so ungrounded answers are visibly missing a
citation; (5) at the monitoring layer, tracking hallucination rate with an evaluation framework like
Ragas (covered in depth in Course 02 — Model Risk Monitoring) to catch regressions after prompt or
model changes.

---

### 4. How do you handle conversation memory at scale?

The model itself is stateless (Chapter 1) — the application owns memory. Practically: store the
running message history per conversation ID in an external store (Redis for low-latency, or Cosmos
DB/Azure SQL for durability), keep compute (App Service/Functions instances) stateless so requests
can be load-balanced across instances without sticky sessions, and manage the token budget as history
grows — either a sliding window (drop oldest turns) or periodic summarization (compress older turns
into a short summary message). See the from-scratch truncation demo in
`notebooks/03_simple_chatbot_with_memory.ipynb` and the discussion in Chapter 3.

---

### 5. LCEL vs. legacy LangChain chains (`LLMChain` etc.) — why does it matter?

Legacy chains each had bespoke APIs. LCEL unifies everything behind the `Runnable` interface
(`invoke`/`batch`/`stream` + async variants), so composing steps with `|` gets you streaming,
batching, and async support automatically, without writing extra code for each chain — directly
useful for a chatbot, where streaming is the difference between a UI that feels instant and one that
feels laggy even at identical total latency. It also composes more predictably (nesting, parallel
branches via `RunnableParallel`, conditional branches via `RunnableBranch`) and integrates directly
with LangServe for deployment. See Chapter 2 for code examples.

---

### 6. What's the difference between calling OpenAI directly and calling Azure OpenAI?

With Azure OpenAI you provision a *deployment* of a specific model version inside your own Azure
OpenAI resource (inside your tenant), and your code calls that deployment name rather than a public
model name — this is the mechanism that keeps data inside the client's compliance boundary, which is
the whole reason a financial-services client would choose Azure OpenAI over the public API. There are
also Azure-specific operational concerns: per-deployment rate limits (TPM/RPM), Azure's own content
filtering layer on top of the model, and API versioning (`api_version` parameter) tied to Azure's
release cadence rather than OpenAI's. See Chapter 3.

---

### 7. How would you evaluate this chatbot before and after a prompt or model change?

Beyond manual spot-checks, a proper evaluation harness would run a held-out set of representative
questions (ideally with known-good answers) through the chain and score dimensions like faithfulness
(is the answer supported by the retrieved context — reduces hallucination false negatives),
relevance, and answer correctness — this is exactly the scope of Course 02 (Model Risk Monitoring),
which covers tools like Ragas for automating these metrics. At minimum for this course's scope: track
response latency, token cost per conversation, and a thumbs-up/down signal collected directly in the
Streamlit/Chainlit UI (Chapter 4) as a lightweight production quality signal between full evaluation
runs.

---

### 8. How would you scale this to 10,000 concurrent users?

Several layers: (1) horizontally scale the stateless API layer (App Service scale-out rules or
Azure Functions consumption/premium plan auto-scaling) since compute holds no session state; (2)
move conversation state to a fast external store (Redis) sized for the concurrency; (3) watch Azure
OpenAI TPM/RPM quotas closely — at that scale you likely need a request queue plus backoff/retry, and
possibly multiple deployments (even multiple Azure OpenAI resources across regions) load-balanced to
raise the effective throughput ceiling; (4) cache aggressively — embeddings for static documents,
and potentially full answers for frequently-asked questions, to cut both cost and load; (5) make
retrieval (Azure AI Search) and generation calls concurrent where the flow allows (`RunnableParallel`
from Chapter 2) rather than always sequential; (6) put Application Insights / equivalent observability
in place early so you can see where the bottleneck actually is (LLM latency vs. retrieval latency vs.
network) before over-engineering a fix.

---

### 9. Why LangChain instead of hand-rolling calls to the Azure OpenAI SDK?

For a single simple call, hand-rolled code is fine. Once you need prompt templating, swappable
backends (Azure OpenAI in prod, a fake/mock model in tests), structured output parsing, conversation
memory, retrieval integration, and streaming — all reused consistently across a codebase —
LangChain's standard abstractions (Chapter 2) remove a lot of repeated glue code and give you a
uniform interface (`Runnable`) that every piece plugs into. The tradeoff is a dependency on the
framework's abstractions and occasional version churn; worth acknowledging if pushed, but the
productivity gain for a multi-feature chatbot backend outweighs it.

---

### 10. What's a system prompt doing for you here that a user prompt can't?

The system prompt sets persistent, hidden-from-the-user behavior — persona, scope constraints, safety
rules, output-format requirements — that applies across the whole conversation, versus the user
prompt which is just the current question. For this chatbot, the system prompt is the primary place
scope-guarding lives ("only answer from provided context; refuse out-of-scope requests"), and it's
also cheaper than fine-tuning as a first lever for controlling behavior. See Chapter 1.

---

### 11. How does the Text-to-Math agent differ architecturally from the main chatbot chain?

The main chatbot is (at its core) a fixed LCEL chain: prompt → LLM → parser, always the same
sequence. The Text-to-Math solver is an **agent** — the LLM decides, at runtime, whether and which
tool to call (a calculator, in this case) based on the ReAct Thought/Action/Observation loop, and
that decision sequence isn't fixed in advance. Use an agent whenever the right sequence of steps
depends on the specific input; use a plain chain when the sequence is always the same. See Chapter 5.

---

### 12. Why route the math evaluation through a calculator tool instead of just asking the LLM for the number?

Because an LLM is a next-token predictor with no arithmetic unit (Chapter 1) — it can produce
plausible-looking numeric answers that are simply wrong on multi-digit arithmetic. Using the LLM for
what it's good at (parsing a word problem's language, deciding the right operations) and a
deterministic Python function for what it's good at (the actual computation) is the general pattern:
let the model reason and delegate, don't ask it to *be* the calculator. See Chapter 5.

---

### 13. What role does Azure DevOps play, and why does it matter for this kind of client?

Azure DevOps hosts the repo and runs the YAML build/release pipeline that tests and promotes changes
(prompt tweaks, code changes, model/deployment version bumps) through dev/test/prod. For a
regulated-industry client, this isn't just convenience — it's the audit trail and control gate that
lets a consultancy demonstrate changes were tested and approved before reaching production, which
matters a great deal when the "production" system is answering questions inside a bank or utility.
Course 04 (CI/CD Pipelines) goes much deeper on this if asked to elaborate.

---

### 14. What would you change if you rebuilt this today?

A few concrete things: First, I'd design conversation memory and RAG grounding in from day one rather
than bolting them on, since token-budget and retrieval-quality issues are much cheaper to fix in the
initial architecture than after the pilot has real users. Second, I'd build an evaluation harness
(Ragas-style faithfulness/relevance scoring, covered in Course 02) alongside the chatbot from the
start rather than treating monitoring as a separate later project — "how do you know it got better"
should be answerable from day one. Third, I'd lean more heavily on Azure OpenAI's native structured
output / function-calling mode rather than prompt-engineered JSON formatting, since it's more
reliable and reduces parser-failure edge cases. And I'd seriously evaluate a smaller/cheaper model
(similar to the Gemma2-9b-on-Groq choice in the Text-to-Math project) for simpler sub-tasks like
query classification or intent routing, reserving the larger Azure OpenAI deployment for final answer
generation — a mixed-model architecture usually beats a single model handling every step at both
cost and latency.

---

### 15. This chatbot served HSBC in production — how did you make sure it could handle real daily customer/employee traffic without hindrance?

"Without hindrance" breaks down into reliability, latency, and headroom, and each has a concrete
lever. Reliability: the compute layer (App Service) is stateless (Chapter 3) so it scales out
horizontally behind the platform's own load balancing, and releases go out via **deployment slots** —
warm the new build in staging, run smoke tests, slot-swap into production — so a release never causes
a downtime window, and a bad release swaps back just as fast. Latency: streaming responses so
time-to-first-token (not total generation time) is what the user actually feels, caching for
frequently-asked questions/embeddings, and watching the retrieval leg (Azure AI Search) and the
generation leg (Azure OpenAI) separately in Application Insights so a slowdown can be attributed to
the right stage rather than guessed at. Headroom: watching Azure OpenAI's per-deployment TPM/RPM
quotas proactively rather than reactively, with exponential backoff/retry and (if traffic justified
it) multiple deployments to raise the effective ceiling. And because this was customer-facing
production traffic rather than an internal pilot, all of that had to be true from day one, not
retrofitted after the first real spike — see the system-design answer on scaling to 10,000 concurrent
users (Q8) for the deeper mechanics.

---

### 16. How would you ensure banking-grade security for a customer-facing LLM chatbot — what does "production-grade" mean here beyond just working code?

"Working code" answers "does it return a correct response." Production-grade for a bank answers "is
every request, and every conversation, defensible to an auditor and unreachable by anyone who
shouldn't have it" — and that's a different set of concerns layered on top of the RAG/prompt
engineering substance (Chapter 3's "Running This in Production for a Bank" section):

1. **Edge protection** — Azure Front Door/Application Gateway terminating TLS and running a WAF in
   front of everything, so raw application compute is never directly exposed to the internet.
2. **Private networking** — the App Service lives inside a VNet and reaches Azure OpenAI, Azure AI
   Search, and Key Vault only through Private Endpoints; nothing backend has a public IP.
3. **Identity, not open access** — Azure AD-based authentication via **MSAL**-acquired **OAuth
   2.0/OIDC** tokens on every request, with **RBAC** enforced server-side on the token's role claims,
   so access maps to real, revocable identities and permissions rather than a shared secret or nothing
   at all.
4. **Secrets out of code** — Key Vault for connection strings/API keys, accessed via managed identity,
   never hardcoded or in application config.
5. **Full audit trail** — Application Insights tracing per conversation: who asked, what was
   retrieved, what model/deployment answered, latency and token counts, and the content-filter
   outcome — the record compliance needs after the fact, not just an uptime dashboard.
6. **Content safety** — Azure OpenAI's built-in content filter plus a scope-guarding system prompt
   (Chapter 1), so the bot has a second line of defense beyond "the model probably behaves."

The theme across all six: nothing about "production-grade" here is about the LLM being smarter — it's
about every layer around the LLM behaving like it belongs inside a bank's compliance boundary.

---

### 17. If Capco later needed to stand up the same chatbot pattern for Bank of America, how would you avoid HSBC's data or configuration leaking into Bank of America's deployment?

The safest and simplest answer is **no shared runtime boundary at all** — separate Azure subscriptions
(or at minimum separate resource groups and VNets) per client, each with its own Azure OpenAI
resource/deployment, its own Azure AI Search index, its own Key Vault, and its own Azure AD tenant/app
registration for auth. That means there's no code path, no shared database, and no shared vector index
where a query or document from one bank could ever be retrieved into the other bank's conversation —
isolation is structural, not just access-controlled. What *can* be shared safely is the
**non-data-bearing layer**: the application code itself (the LangChain/LCEL pipeline, the UI, the
Azure DevOps pipeline templates) can be the same reusable codebase deployed twice, parameterized by
environment/config per client, as long as the deployment pipeline itself is strict about which
config/secrets go to which environment (separate service connections and variable groups per client
in Azure DevOps, so a HSBC-targeted pipeline run has no credentials or endpoints for the Bank of
America environment even by mistake). The failure mode to actively design against is a shared
multi-tenant service with a `client_id` column or a shared vector index namespaced by tenant — it's
cheaper to build, but it turns "no cross-tenant leakage" from a structural guarantee into an
application-logic guarantee, which is a much easier thing to get wrong at a bank's risk tolerance.

---

### 18. What's different about debugging a production incident in this chatbot versus a bug in a batch ML pipeline?

A batch ML pipeline failure is usually reproducible and bounded: you have the exact input dataset, the
job either failed loudly (stack trace, exit code) or produced a wrong output you can diff against
expected, and you can re-run it as many times as you want against the same inputs without affecting
anyone. A production chatbot incident is live, ongoing, and often not cleanly reproducible: a user is
mid-conversation right now, the failure might be an intermittent 429 from Azure OpenAI, a
content-filter false positive, a retrieval miss that only happens for a specific phrasing, or a
latency spike from a noisy-neighbor effect on a shared resource — and you're debugging it under time
pressure while traffic keeps arriving. Practically that changes the workflow: you lean on **Application
Insights' per-conversation tracing** (Chapter 3) to reconstruct exactly what happened for the affected
user — which retrieval results came back, which model/deployment answered, what the latency breakdown
was — rather than re-running a job locally; you think in terms of **mitigate first, root-cause
second** (e.g., a deployment-slot swap back to the last known-good build buys time to investigate
without leaving users in a broken state); and you have to account for the LLM's non-determinism itself
— the same prompt won't always reproduce the same output, so "it worked when I tried it just now"
isn't evidence the incident is resolved. Course 10 (Toughest Challenges) goes deeper on production
incident patterns across all the projects in this course set.

---

### Quick follow-up probes to be ready for

- "What's `top_p` vs temperature, concretely?" → Chapter 1.
- "What does `RunnableWithMessageHistory` actually do?" → Chapter 2 / Chapter 3.
- "What happens if Azure AI Search returns no relevant chunks?" → the system prompt should have the
  model explicitly say it doesn't know rather than guessing — Chapter 1 and Chapter 3.
- "How would you test an agent's tool-calling reliability?" → held-out set of problems with known
  answers, assert both the tool was called and the final numeric answer matches — Chapter 5.
