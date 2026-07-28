# 99 — Interview Q&A: Capco AI Chatbot Assistant

Read this last, after the chapters and notebooks are fresh. Practice the behavioral answer out loud
first — it's your opener for almost any "tell me about a project" prompt. **Q2 is a near-certain
follow-up once an interviewer hears this is a RAG-grounded, production chatbot** — it's the natural
sequel to "how do you keep answers accurate," and it's positioned right up front, deliberately, the
same way course 05's document-versioning question leads that course's Q&A. **Q3 is a real question the
candidate was actually asked in an interview** ("what if the user asks vague questions that drift away
from the chatbot's context and capability?"), followed immediately by the real, actually-asked follow-up
("what kind of guardrails?") — it's featured right after Q2 for the same reason: both are the kind of
question a one-line answer doesn't survive.

---

### 1. (Behavioral) Tell me about this project.

Use the STAR summary from `00-README.md` verbatim, practiced to under 90 seconds:

> At Capco, a financial-services client's support and ops teams were losing time manually searching
> internal documentation to answer routine questions. I was asked to build an intelligent chatbot
> that could understand natural-language questions and return accurate, grounded answers, entirely
> within the client's Azure environment. I built the backend in Python using Azure OpenAI as the LLM
> provider, composed the pipeline with LangChain/LCEL, added prompt engineering and conversation
> memory to keep answers accurate and context-aware, and built guardrails to handle vague or
> out-of-scope questions gracefully. The production interface was `react-service`, a dedicated React
> frontend microservice calling the backend over a streaming API and authenticating through Azure
> AD/MSAL (I used Streamlit/Chainlit for faster internal prototyping along the way, not the production
> UI), and I set up the Azure DevOps CI/CD pipeline so changes deployed safely through dev/test/prod.
> *(Illustrative)* The pilot cut query resolution time by roughly 35% and deflected about 40% of
> routine support tickets.

Remember to swap the italicized result for your real numbers before using this live.

---

### 2. How do you handle it when the source documents behind this chatbot's answers get updated or corrected?

This is the gotcha question — the direct RAG-chatbot analog of the document-versioning question in
course 05, and a natural follow-up the moment an interviewer hears "grounded, RAG-based, production."
The full worked answer lives in Chapter 6
(`06-knowledge-freshness-and-conversation-state-lifecycle.md`) — this is the compressed,
interview-ready version:

> "Today, a realistic setup like this refreshes the retrieval index on a schedule, not on a document
> change — there's no event anywhere that says 'this specific document was just superseded.' And
> because the document-ingestion side (course 05's Document Uploader Service) gives every upload,
> including a revision, a brand-new document ID with no supersede link, the index has no way to know
> two sets of chunks belong to the same logical document — a revision doesn't replace the old chunks,
> it just adds new ones next to them, and retrieval can return either, or both, with no signal about
> which is current. On top of that, if there's a full-answer cache for frequently-asked questions, it
> has no idea a source document changed either — it's just keyed on the question text with a timer,
> so it can keep serving a cached wrong answer even after the index itself gets fixed.
>
> What you can do about that today, with existing Azure AI Search capabilities, is a manual but real
> workaround: get the stale document's ID (which falls out naturally from the same search course 05's
> upload-side workaround already does), delete its chunks from the index with a delete-by-filter call
> on that document ID, force an out-of-cycle index refresh, and manually clear the answer cache if one
> exists. All three operations genuinely exist and work today — the gap is that nothing triggers them
> automatically.
>
> It's also worth being precise that this is a completely different concern from conversation memory.
> Clearing a user's conversation history doesn't fix a stale answer — a user with zero prior turns in
> a brand-new session gets exactly the same stale answer as someone forty turns into a long
> conversation, because the staleness lives in the index and the cache, not in anything
> session-scoped. Those are two independent axes and conflating them is a real mistake to avoid.
>
> If I were building a real fix, I'd make it event-driven instead of scheduled: the moment the
> uploader service records that document B supersedes document A — assuming it had the
> `supersedes_document_id` field course 05 proposes but doesn't have today — that write should publish
> an event a small worker consumes to immediately delete the old chunks from the index, evict any
> cached embeddings for that document, and evict any cached full answers that cited it. And I'd design
> the caches to fail safe even if that event is ever missed: key the embedding cache on content hash
> plus model version so a changed chunk naturally gets a new key instead of serving old content under
> the same one, and keep the answer cache's TTL short — minutes, not hours — as the fallback safety net
> behind the event. The honest scope of that fix reaches three systems, not one: the uploader service
> has to emit the signal, the search index has to consume it, and the cache layer has to consume it
> too — fixing only the index without the cache just moves where the stale answer comes from."

**Follow-ups to be ready for (the first three lean on Chapter 7):**

- *"Doesn't horizontal scaling make the cache-invalidation problem worse?"* — Yes, directly: if that
  answer cache (or a system-prompt/config cache) is a plain in-process dict rather than a shared store
  like Redis, every App Service instance builds its own independent cache, so an invalidation event has
  to reach every instance individually — a stale answer can keep being served from whichever instance
  the load balancer happens to route a user to, even after other instances have already purged it
  (Chapter 7's scaling/concurrency caveat).
- *"What's the cheapest thing you could do today, without an event system, to bound the damage?"* —
  Shorten the answer cache's TTL aggressively (minutes, not hours) so a missed invalidation self-heals
  quickly regardless of whether the event-driven fix ever ships, and surface an `indexed_as_of`
  timestamp on each citation so a reviewer can visually judge whether an answer might be running on
  stale grounding rather than trusting it blindly.
- *"How would you even detect this was happening in production, before a user or reviewer noticed?"* —
  An automated freshness regression test: ask a canonical question, simulate a document-update event
  for the source it should be grounded in, ask the identical question again, and assert the answer
  actually changed — the same test that would have caught the stale-answer-cache bug in Chapter 7
  before a compliance reviewer found it manually.
- *"Is this the same as IWPB's approval workflow you might have described for the uploader service?"* —
  No, and worth saying so plainly if it comes up: an approval/retention lifecycle for one uploaded
  document instance is a different axis entirely from whether the *retrieval and cache layers* built on
  top of that document are currently serving a current or superseded version of it — the same
  distinction course 05's chapter 05 draws for its own project.

---

### 3. What if the user asks vague questions that drift away from the chatbot's context and capability? / What kind of guardrails did you build?

This is a real question-and-follow-up the candidate faced in an interview. The first answer given —
"we have built guardrails to deal with vague and out-of-context questions" — is true but thin, and the
interviewer's natural next question, actually asked, was "what kind of guardrails?" The full worked
answer lives in Chapter 08 (`08-guardrails-scope-and-vague-query-handling.md`); this is the
compressed, interview-ready version, aimed at under 90 seconds spoken:

> "The first thing worth separating out is that 'guardrails' actually covers three different problems.
> There's a vague-but-plausibly-in-scope question — something like 'tell me about the policy' with no
> policy named — where the question is ambiguous, not wrong. There's a genuinely out-of-scope
> question — asking an internal HR/IT assistant about the weather, say — where it's just not something
> the bot should attempt. And there's an adversarial question — someone trying prompt injection or a
> jailbreak — which is a security problem, not a scope problem, and I'd point to the bad-actor testing
> we covered separately rather than treat it the same way.
>
> For the first two, the architecture is layered. The system prompt scopes the bot's role and gives it
> a refusal template. Before generation even runs, a retrieval-confidence gate checks whether RAG
> actually found relevant content above a similarity threshold — if it didn't, we short-circuit to a
> fallback instead of letting the model free-generate and risk hallucinating. Ahead of that, a
> lightweight, cheap classification step — an embedding-similarity check against a set of reference
> in-scope topics — runs before the expensive retrieval-and-generation call and decides which of three
> paths the query takes. If it's vague-but-in-scope, the bot asks a targeted clarifying question rather
> than guessing — and if the frontend supports it, that clarification renders as quick-reply buttons
> instead of forcing the user to type a follow-up. If it's out-of-scope, the bot gives a scoped refusal
> that also reminds the user what it can actually help with, which is a much better experience than a
> bare 'I can't answer that.'
>
> One distinction I'd flag explicitly: content-safety filtering and topical-scope filtering are not the
> same guardrail. Azure OpenAI's built-in content filter catches harmful content — hate, violence,
> self-harm — independent of topic. Scope filtering answers a completely different question: is this
> something the bot is even supposed to be answering. A question can pass content-safety cleanly and
> still be totally out of scope, and conflating the two is an easy mistake.
>
> And it's not single-turn only — a conversation can start on-topic and drift over several turns in a
> way no single message would flag on its own, so there's session-level topic tracking watching the
> conversation as a whole, not just each message in isolation.
>
> None of this is a one-and-done classifier, either — the harmfulness, bad-actor, and robustness
> monitoring we built separately is exactly what would catch these guardrails degrading in production,
> so it's continuously monitored, not set-and-forget."

**Three sharp follow-ups to be ready for:**

- *"How do you handle a query that's vague but could plausibly still be in-scope, versus genuinely
  out-of-scope?"* — Both get caught by the same pre-generation classification step, but the confidence
  band differs: a vague-but-in-scope query has *some* similarity signal to a known topic, just not
  enough to commit to one confidently, so it earns a clarifying question. A genuinely out-of-scope query
  has no meaningful similarity to any in-scope topic at all, so it earns a scoped refusal instead of a
  clarification — asking "which of our topics is this about?" doesn't make sense for a question that
  isn't about any of them. See Chapter 08, Part 2, items 3–5, and the runnable three-way split in
  `notebooks/08_scope_and_vague_query_classifier_demo.ipynb`.
- *"How would you avoid an annoying back-and-forth clarification loop?"* — Two things: first, the
  clarification is targeted, not generic — it's built from the actual candidate topics the classifier
  scored close to, not a blanket "can you rephrase that?" Second, this is exactly the place `react-service`
  (Chapter 04) earns its keep as a frontend, not just a display layer — rendering the clarifying question
  as quick-reply/suggested-question buttons turns a potential multi-message back-and-forth into a single
  tap, which is a meaningfully different user experience than the same clarification as plain text the
  user has to type a reply to.
- *"Is content filtering the same thing as scope/topicality filtering?"* — No, and this is worth stating
  plainly rather than letting the interviewer assume they're the same thing: content-safety filtering
  (Azure OpenAI's built-in filter, Chapter 03) asks "is this harmful, independent of topic" — hate,
  violence, self-harm, sexual content. Scope/topicality filtering asks a completely different question —
  "is this something the bot is supposed to answer at all." A question can be perfectly safe and still be
  wildly out of scope, and the two are checked by entirely different mechanisms (a platform-level content
  filter vs. the system-prompt/retrieval-gate/classifier stack in Chapter 08) — conflating them is a
  common mistake worth naming and avoiding.

---

### 4. Walk me through what happens, end to end, when a user sends a message to the chatbot.

Message arrives at the Python API layer (Azure App Service/Functions) → the backend pulls the
user's conversation history from the session store → (if RAG-enabled) the question is embedded and
used to query Azure AI Search for relevant document chunks → those chunks plus the history are
inserted into a `ChatPromptTemplate` → the LCEL chain (`prompt | llm | parser`) invokes Azure OpenAI
→ the response streams token-by-token back through the API to the UI, where it renders incrementally
→ the full exchange is appended to the session store and logged for audit/monitoring. See
`00-README.md`'s architecture diagram and Chapter 3 for the full picture.

---

### 5. How would you reduce hallucination in this chatbot?

Layered defenses, from cheapest to most involved: (1) a tight system prompt that explicitly instructs
the model to answer only from provided context and to say "I don't know" otherwise (Chapter 1); (2)
low temperature (0–0.3) so answers are consistent rather than creative; (3) grounding via RAG —
retrieving the actual relevant policy text via Azure AI Search and inserting it into the prompt, so
the model has the correct facts in context rather than relying on parametric memory (Chapter 3); (4)
structured output with a required "source" field, so ungrounded answers are visibly missing a
citation; (5) at the monitoring layer, tracking hallucination rate with an evaluation framework like
Ragas (covered in depth in Course 04 — Model Risk Monitoring) to catch regressions after prompt or
model changes.

---

### 6. How do you handle conversation memory at scale?

The model itself is stateless (Chapter 1) — the application owns memory. Practically: store the
running message history per conversation ID in an external store (Redis for low-latency, or Cosmos
DB/Azure SQL for durability), keep compute (App Service/Functions instances) stateless so requests
can be load-balanced across instances without sticky sessions, and manage the token budget as history
grows — either a sliding window (drop oldest turns) or periodic summarization (compress older turns
into a short summary message). See the from-scratch truncation demo in
`notebooks/03_simple_chatbot_with_memory.ipynb` and the discussion in Chapter 3. Worth distinguishing
explicitly from a related-sounding but different problem: conversation-memory management governs how
much of the *dialogue* the model sees, while whether the *retrieved content* itself is current is a
completely separate axis — Chapter 6 covers that distinction in depth if pressed on it.

---

### 7. LCEL vs. legacy LangChain chains (`LLMChain` etc.) — why does it matter?

Legacy chains each had bespoke APIs. LCEL unifies everything behind the `Runnable` interface
(`invoke`/`batch`/`stream` + async variants), so composing steps with `|` gets you streaming,
batching, and async support automatically, without writing extra code for each chain — directly
useful for a chatbot, where streaming is the difference between a UI that feels instant and one that
feels laggy even at identical total latency. It also composes more predictably (nesting, parallel
branches via `RunnableParallel`, conditional branches via `RunnableBranch`) and integrates directly
with LangServe for deployment. See Chapter 2 for code examples.

---

### 8. What's the difference between calling OpenAI directly and calling Azure OpenAI?

With Azure OpenAI you provision a *deployment* of a specific model version inside your own Azure
OpenAI resource (inside your tenant), and your code calls that deployment name rather than a public
model name — this is the mechanism that keeps data inside the client's compliance boundary, which is
the whole reason a financial-services client would choose Azure OpenAI over the public API. There are
also Azure-specific operational concerns: per-deployment rate limits (TPM/RPM), Azure's own content
filtering layer on top of the model, and API versioning (`api_version` parameter) tied to Azure's
release cadence rather than OpenAI's. See Chapter 3.

---

### 9. How would you evaluate this chatbot before and after a prompt or model change?

Beyond manual spot-checks, a proper evaluation harness would run a held-out set of representative
questions (ideally with known-good answers) through the chain and score dimensions like faithfulness
(is the answer supported by the retrieved context — reduces hallucination false negatives),
relevance, and answer correctness — this is exactly the scope of Course 04 (Model Risk Monitoring),
which covers tools like Ragas for automating these metrics. At minimum for this course's scope: track
response latency, token cost per conversation, and a thumbs-up/down signal collected directly in the
production `react-service` UI (Chapter 04) as a lightweight production quality signal between full
evaluation runs — the same feedback-capture idea Chainlit/Streamlit's built-in widgets gave for free
during the earlier prototyping phase, just implemented as a first-class part of the production frontend.

---

### 10. How would you scale this to 10,000 concurrent users?

Several layers: (1) horizontally scale the stateless API layer (App Service scale-out rules or
Azure Functions consumption/premium plan auto-scaling) since compute holds no session state; (2)
move conversation state to a fast external store (Redis) sized for the concurrency; (3) watch Azure
OpenAI TPM/RPM quotas closely — at that scale you likely need a request queue plus backoff/retry, and
possibly multiple deployments (even multiple Azure OpenAI resources across regions) load-balanced to
raise the effective throughput ceiling; (4) cache aggressively — embeddings for static documents,
and potentially full answers for frequently-asked questions, to cut both cost and load, while being
deliberate about how that cache is invalidated (Chapter 6) so scale doesn't just mean serving stale
answers faster; (5) make retrieval (Azure AI Search) and generation calls concurrent where the flow
allows (`RunnableParallel` from Chapter 2) rather than always sequential; (6) put Application Insights
/ equivalent observability in place early so you can see where the bottleneck actually is (LLM latency
vs. retrieval latency vs. network) before over-engineering a fix; (7) be aware that "stateless compute"
doesn't automatically mean *everything* in the process is shared across instances — an in-process
cache or config value can quietly reintroduce per-instance state even in an otherwise horizontally
scaled design (Chapter 7).

---

### 11. Why LangChain instead of hand-rolling calls to the Azure OpenAI SDK?

For a single simple call, hand-rolled code is fine. Once you need prompt templating, swappable
backends (Azure OpenAI in prod, a fake/mock model in tests), structured output parsing, conversation
memory, retrieval integration, and streaming — all reused consistently across a codebase —
LangChain's standard abstractions (Chapter 2) remove a lot of repeated glue code and give you a
uniform interface (`Runnable`) that every piece plugs into. The tradeoff is a dependency on the
framework's abstractions and occasional version churn; worth acknowledging if pushed, but the
productivity gain for a multi-feature chatbot backend outweighs it.

---

### 12. What's a system prompt doing for you here that a user prompt can't?

The system prompt sets persistent, hidden-from-the-user behavior — persona, scope constraints, safety
rules, output-format requirements — that applies across the whole conversation, versus the user
prompt which is just the current question. For this chatbot, the system prompt is the primary place
scope-guarding lives ("only answer from provided context; refuse out-of-scope requests"), and it's
also cheaper than fine-tuning as a first lever for controlling behavior. See Chapter 1. It's also
worth knowing what happens when this goes wrong operationally, not just conceptually: Chapter 7 walks
through a realistic bug where a truncation refactor silently evicted the system prompt itself on long
conversations, and a separate caveat about system-prompt version skew across a scaled-out fleet during
a rolling deployment — the system prompt being "always there" is a design intent, not a guarantee, and
it's worth being able to say what could break it.

---

### 13. How does the Text-to-Math agent differ architecturally from the main chatbot chain?

The main chatbot is (at its core) a fixed LCEL chain: prompt → LLM → parser, always the same
sequence. The Text-to-Math solver is an **agent** — the LLM decides, at runtime, whether and which
tool to call (a calculator, in this case) based on the ReAct Thought/Action/Observation loop, and
that decision sequence isn't fixed in advance. Use an agent whenever the right sequence of steps
depends on the specific input; use a plain chain when the sequence is always the same. See Chapter 5.

---

### 14. Why route the math evaluation through a calculator tool instead of just asking the LLM for the number?

Because an LLM is a next-token predictor with no arithmetic unit (Chapter 1) — it can produce
plausible-looking numeric answers that are simply wrong on multi-digit arithmetic. Using the LLM for
what it's good at (parsing a word problem's language, deciding the right operations) and a
deterministic Python function for what it's good at (the actual computation) is the general pattern:
let the model reason and delegate, don't ask it to *be* the calculator. See Chapter 5.

---

### 15. What role does Azure DevOps play, and why does it matter for this kind of client?

Azure DevOps hosts the repo and runs the YAML build/release pipeline that tests and promotes changes
(prompt tweaks, code changes, model/deployment version bumps) through dev/test/prod. For a
regulated-industry client, this isn't just convenience — it's the audit trail and control gate that
lets a consultancy demonstrate changes were tested and approved before reaching production, which
matters a great deal when the "production" system is answering questions inside a bank or utility.
Course 06 (CI/CD Pipelines) goes much deeper on this if asked to elaborate.

---

### 16. What would you change if you rebuilt this today?

A few concrete things: First, I'd design conversation memory and RAG grounding in from day one rather
than bolting them on, since token-budget and retrieval-quality issues are much cheaper to fix in the
initial architecture than after the pilot has real users. Second, I'd build an evaluation harness
(Ragas-style faithfulness/relevance scoring, covered in Course 04) alongside the chatbot from the
start rather than treating monitoring as a separate later project — "how do you know it got better"
should be answerable from day one. Third, I'd lean more heavily on Azure OpenAI's native structured
output / function-calling mode rather than prompt-engineered JSON formatting, since it's more
reliable and reduces parser-failure edge cases. Fourth, I'd design knowledge-freshness as an
event-driven pipeline from the start (Chapter 6) — an uploader-service supersede event flowing through
to a search-index purge and a cache invalidation — rather than a scheduled-refresh-plus-manual-cleanup
setup that only gets revisited once someone notices a stale answer. And I'd seriously evaluate a
smaller/cheaper model (similar to the Gemma2-9b-on-Groq choice in the Text-to-Math project) for simpler
sub-tasks like query classification or intent routing, reserving the larger Azure OpenAI deployment for
final answer generation — a mixed-model architecture usually beats a single model handling every step
at both cost and latency.

---

### 17. This chatbot served HSBC in production — how did you make sure it could handle real daily customer/employee traffic without hindrance?

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
users (Q10) for the deeper mechanics, and Chapter 7 for the realistic error-handling table behind
"reliability" above.

---

### 18. How would you ensure banking-grade security for a customer-facing LLM chatbot — what does "production-grade" mean here beyond just working code?

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
about every layer around the LLM behaving like it belongs inside a bank's compliance boundary. Chapter
7 extends this same theme downward into the operational layer: production-grade also means a
deliberately asymmetric error-handling table (degrade gracefully on a retrieval outage, fail loud on
anything unrecognized) and honestly-named limits like per-instance caches under a scaled-out fleet.

---

### 19. This chatbot runs on multiple App Service instances — what's the concrete scaling caveat you'd name, if asked?

Not conversation history — that already lives in an external session store shared identically across
every instance. The real caveat is whatever's still sitting in a plain in-process cache on a single
worker: a full-answer cache for frequently-asked questions, and a system-prompt/config cache loaded
once at process startup. Neither is shared across instances by default. Concretely: **cache hit rate
fragments** (a question cached on instance A is a cold miss on instance B seconds later), **cache
invalidation doesn't propagate** (Chapter 6's proposed event-driven purge has to reach every instance
individually, or a stale answer can keep surfacing from whichever instance the load balancer happens to
route a user to), and — a distinct failure shape worth naming separately — **prompt/config version skew
during a rolling deployment**, where instances mid-rollout are simultaneously serving old and new
system-prompt versions, which can look like "the model behaving inconsistently" when the actual cause
is fleet-level config skew, not the model. Fixes are concrete, not hand-wavy: move both caches to a
shared store (Redis) with a short TTL or push-based invalidation, or accept session affinity at the
load balancer as the cheaper interim fix. Chapter 7 has the full detail.

---

### 20. Tell me about a bug you found and fixed that was specific to building a GenAI product, not just general backend work.

Pick one of the four from Chapter 7 — the streaming one is a strong, concrete choice because it's a
genuinely GenAI-specific failure mode, not a generic backend bug: the backend's SSE streaming
endpoint (consumed by the `react-service` frontend) checked
`finish_reason == "stop"` and broke out of the chunk loop before appending that same chunk's
`delta.content`, on the (usually true but not always) assumption that the terminal chunk carries no
content of its own. In a minority of completions, Azure OpenAI packed the last visible token together
with the stop signal in a single chunk, so that last word — sometimes a citation label — was silently
dropped from what the user saw *and* from what got persisted to conversation history, meaning every
later turn's context was quietly missing it too. Nothing threw an exception; the response still
rendered, just truncated by one token — exactly the "looks fine, isn't" class of bug that only a
correctness-focused test catches. What would have caught it: a streaming-equivalence test — assert that
concatenating every streamed chunk for a fixed prompt produces byte-identical output to the same
prompt's non-streaming response, across enough sample completions to catch a low-frequency ordering
issue rather than relying on a handful of manual spot checks.

---

### 21. What timeout and retry values would you set for calls to Azure OpenAI, and why?

Concrete numbers worth having ready (Chapter 7, illustrative but specific): a short connect timeout
(~5s, since a slow connection almost always means a network/DNS problem worth failing fast on) paired
with a much longer read timeout for streaming calls (~60s, since a slow *response* on an already-open
connection can just mean a long completion at a busy moment); retries capped at 3 attempts with
exponential backoff (2s/4s/8s, jittered, capped around 20s) restricted to a narrow allowlist of
genuinely transient failures (429/500/502/503/504) — never a blanket "retry anything that raises,"
because retrying an unrecognized failure is exactly how you risk a double-call; and preferring the
API's own `Retry-After` header over a computed backoff on 429s, since Azure OpenAI's quota system knows
its own reset window better than a client-side guess. The retry-scope point is worth making
proactively: retrying is only safe if you can prove the failed call never got a response started — a
client-side timeout that fires *after* a stream has already begun is not safe to blindly retry, and
getting that distinction wrong is exactly the kind of double-billing, interleaved-response bug Chapter
7 walks through.

---

### 22. If Capco later needed to stand up the same chatbot pattern for Bank of America, how would you avoid HSBC's data or configuration leaking into Bank of America's deployment?

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

### 23. What's different about debugging a production incident in this chatbot versus a bug in a batch ML pipeline?

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
isn't evidence the incident is resolved. Course 12 (Toughest Challenges) goes deeper on production
incident patterns across all the projects in this course set.

---

### Quick follow-up probes to be ready for

- "What's `top_p` vs temperature, concretely?" → Chapter 1.
- "What does `RunnableWithMessageHistory` actually do?" → Chapter 2 / Chapter 3.
- "What happens if Azure AI Search returns no relevant chunks?" → the system prompt should have the
  model explicitly say it doesn't know rather than guessing — Chapter 1 and Chapter 3.
- "How would you test an agent's tool-calling reliability?" → held-out set of problems with known
  answers, assert both the tool was called and the final numeric answer matches — Chapter 5.
- "How is 'the conversation feels stale' different from 'the answer is factually stale'?" → the first
  is a token-budget/memory-management question (Chapter 3); the second is an index/cache-freshness
  question that a fresh, zero-turn conversation is just as exposed to (Chapter 6) — don't reach for the
  memory-management answer when the real issue is retrieval grounding.
- "Name one thing that's per-process in an otherwise stateless-sounding architecture." → an in-process
  answer cache or system-prompt/config cache, both discussed with concrete fixes in Chapter 7.
