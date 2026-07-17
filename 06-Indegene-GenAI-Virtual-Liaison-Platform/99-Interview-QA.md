# 99 — Interview Q&A: Indegene GenAI Virtual Liaison Platform

This bank is larger than other courses' — this project has more moving sub-components (RAG x2,
summarization, entity extraction, recommenders, multilingual routing), plus real two-client production
deployment questions (Eli Lilly and AstraZeneca, live isolation, ECS Fargate), so expect interviewers
to probe each one individually as well as the system as a whole. Work through these after all seven
chapters; each answer references the chapter with the full depth.

## Behavioral (STAR-based)

### Q1. Tell me about a project where you built something end-to-end, from ambiguous requirements to a working system.

**A.** Use the STAR summary in `00-README.md` almost verbatim: Situation (liaisons spending time on
repetitive project questions across languages), Task (build a GenAI assistant clients could chat/talk
to directly), Action (RAG over project data and cost catalog with LangChain + Pinecone, summarization,
entity extraction for new requests, recommenders, all behind a multilingual chat/voice layer), Result
(illustrative: scoping turnaround cut from ~3 days to under 1 day, 30-40% of routine questions
deflected from human liaisons). Practice it under 90 seconds — this project has five sub-features, so
the temptation is to over-explain each one; resist that in the STAR answer and save the depth for
follow-ups.

### Q2. Describe a time you had to make an architectural trade-off under time or resource constraints.

**A.** A strong real answer from this project: choosing embedding-similarity recommenders (Chapter 6)
over collaborative filtering as the primary recommendation mechanism, specifically because
collaborative filtering needs substantial interaction history to work and a newly launched internal
platform doesn't have that yet — the cold-start problem. Reusing the same embedding + Pinecone
infrastructure already built for RAG meant no separate recommendation pipeline had to be built and
maintained, which was both a technical and a resourcing trade-off. Name the trade-off explicitly:
collaborative filtering could eventually surface non-obvious associations embedding similarity can't,
but wasn't viable on day one.

### Q3. Tell me about a time you had to explain a technical decision to a non-technical stakeholder (e.g., why the assistant said what it said).

**A.** RAG's grounding + citation model (Chapter 1) is built for exactly this: because every answer is
generated from retrieved, loggable chunks rather than the model's opaque parametric knowledge, you can
show a client or reviewer *which* project-status note or cost-catalog entry backed a given answer.
Frame the story around a moment where that traceability mattered — e.g., a client questioning a cost
figure, and being able to point to the exact catalog line item retrieved, rather than saying "the AI
said so."

## Technical Deep-Dive — Per Sub-Feature

### Q4. Why did you use RAG instead of fine-tuning a model on Indegene's project data?

**A.** See Chapter 1. Project data changes daily (fine-tuning would be perpetually stale), fine-tuning
doesn't give clean per-client data isolation (facts get blended into weights, not separable at
inference time), and RAG makes freshness a matter of writing one new record, plus makes isolation a
retrieval-time filter (Pinecone namespace/metadata) rather than a training-time hope. Fine-tuning is
the right tool for changing model *behavior* (tone, format, task-following), not for injecting
frequently-changing *facts*.

### Q5. How do you keep RAG answers grounded when the user asks in a different language than the source documents?

**A.** This is a Virtual-Liaison-specific question worth having a crisp answer for, since the resume
explicitly calls out multilingual interaction. The practical design: normalize the client's message to
a common working language (e.g., English) via the language-detection/translation layer *before* it
reaches retrieval (Chapter 7's "where voice and translation fit" section) — so embeddings, BM25
tokenization, and the grounding prompt all operate on one consistent language, rather than needing a
separate retrieval index or embedding model per client language. The response is generated in that
working language, grounded in the (also-working-language, or already-multilingual-embedded) source
documents, then translated back to the client's language as the last step. The risk to name
explicitly: translation is a lossy step on both ends, and a translation error before retrieval can
cause a wrong or missed retrieval that's invisible downstream — worth mentioning that using a
multilingual-native embedding model (embedding source documents and queries directly in their
original languages into a shared multilingual embedding space) is an alternative that avoids the
pre-translation step entirely, at the cost of needing an embedding model with genuinely strong
multilingual performance for the specific language pairs in play.

### Q6. Walk me through your chunking strategy for project data, and why it mattered.

**A.** See Chapter 1. Chunk at the natural unit of the data first (one status update, one revision
note, one catalog line) rather than blind fixed-size splitting, because a chunk boundary that cuts a
status update in half produces an incoherent, hard-to-cite retrieval result. Fall back to
recursive/size-based splitting with overlap only for free-text notes that don't have that structure.
Tie it to a concrete failure mode: fixed 500-token chunking with no overlap can split "JP launch
delayed" from "due to legal review" into two different chunks, and a query about the delay reason
might only retrieve the first half.

### Q7. Why Pinecone specifically, and how did you keep different clients' project data isolated?

**A.** See Chapter 2. Pinecone is a managed vector database handling ANN indexing (HNSW), sharding,
and replication, which is a reasonable buy-vs-build choice when the team is optimizing for delivery
speed over owning vector-search infrastructure (vs. e.g. self-hosting FAISS, which has no built-in
persistence, filtering, or multi-tenancy). Isolation: one **namespace per client** as the hard security
boundary (a query against one namespace physically cannot return another namespace's vectors),
**metadata filters** (`project_id`, `doc_type`) for finer relevance filtering within a namespace, and
the namespace value always derived server-side from the authenticated session — never from user input.

### Q8. Hybrid search vs. pure dense retrieval — when does it matter?

**A.** See Chapter 3. It matters when the query distribution includes a meaningful share of exact-
identifier lookups — project codenames, cost-catalog SKUs — that embedding models have weak
representations for, since those tokens are rare/arbitrary strings the model has little learned
meaning for. Dense retrieval alone can rank a semantically-similar-but-wrong codename above the exact
match; BM25 (sparse) finds the exact token but misses paraphrase. Reciprocal rank fusion combines
both rankings by position rather than raw score (avoiding the problem of BM25 and cosine scores living
on different scales), so a chunk that's the literal SKU match and a chunk that's the best semantic
match both surface near the top. For a purely conversational FAQ system with no structured
identifiers, plain dense retrieval is usually sufficient and hybrid search would be unnecessary
operational overhead — know when *not* to reach for it, too.

### Q9. How would you evaluate your RAG pipeline's answer quality?

**A.** Not explicitly built as a chapter here (see course 02's LLM-evaluation-focused project for the
full depth), but the answer worth giving: retrieval quality (precision/recall of retrieved chunks
against a labeled or LLM-judged relevance set) and generation quality (faithfulness — is the answer
actually supported by the retrieved context, i.e. not hallucinated beyond it — plus answer
relevance) are two separate things to measure, and a good-looking final answer can still hide bad
retrieval if the model happens to "know" the answer anyway. For this platform specifically, project-
status and cost-catalog answers are high-stakes enough to warrant human spot-checks against source
records, not just automated metrics, especially early on.

### Q10. Extractive vs. abstractive summarization — why did you choose the one you did for project tracking?

**A.** See Chapter 4. Abstractive (LLM-generated) summarization is the right default for "give me a
concise update" because clients want a synthesized narrative, not a bag of quoted sentences —
extractive summarization can't produce a sentence like "the delay was resolved three updates later"
because that's a synthesis across multiple source sentences, not a verbatim extraction. Extractive
scoring is still useful as a cheap pre-filter to control token cost before the abstractive step,
especially on long histories.

### Q11. A project's history is too long to fit in the model's context window. How do you summarize it?

**A.** See Chapter 4 — map-reduce vs. refine. Map-reduce summarizes chunks independently and combines
them, which parallelizes well but can lose cross-chunk continuity (a delay mentioned in one chunk and
resolved several chunks later doesn't naturally get reconciled). Refine processes chunks sequentially,
updating a running summary with each new chunk, which preserves narrative continuity at the cost of
being strictly sequential (can't parallelize, latency scales with chunk count). For project tracking
specifically — where status evolves over time and continuity matters — refine is generally the better
fit, ideally maintained incrementally (re-refining only with new updates as they arrive) rather than
recomputed from full history on every request.

### Q12. Why LLM-based structured extraction over training a custom NER model for entity extraction?

**A.** See Chapter 5. Rule-based regex only catches phrasing you explicitly anticipated and breaks on
novel wording. A trained spaCy NER model generalizes better but needs labeled training data and
retraining/maintenance as request phrasing evolves, and doesn't natively handle multilingual input
without per-language models. LLM-based structured extraction (Pydantic schema + function calling /
structured output) handles paraphrase and multiple languages in one pass, and extending it to a new
field is a schema/prompt edit, not a data-collection-and-retraining cycle — the right trade for an
evolving, open-ended set of project-request fields. The mitigation for the obvious risk (LLM
hallucinating a value the client never mentioned): optional fields with an explicit "do not invent
values" instruction, plus a lightweight rule-based validation pass on the output before it reaches
project creation.

### Q13. How do you evaluate a recommender system when you have sparse historical interaction data?

**A.** See Chapter 6. Standard offline metrics (precision@K, recall@K, NDCG) need a reasonably large
held-out interaction set that a young or low-volume internal platform doesn't have. Practical
alternatives: proxy relevance labels from content/business-rule similarity (same asset type, same
industry vertical) as an approximate ground truth; human-in-the-loop evaluation from project managers
rating a sample of recommendations; and, most importantly, transitioning to **online** metrics
(click-through/acceptance rate on suggested "similar projects") once the system is live, which also
accumulates the interaction data needed to make collaborative filtering viable later. Say plainly that
sparse-data evaluation is inherently approximate — that's the honest and correct answer, not a gap to
hide.

## System Design

### Q14. Design the routing logic that decides whether a user query needs RAG retrieval, summarization, entity extraction, or a recommendation.

**A.** See Chapter 7's "Designing the Router" section. Two layers: a cheap rule/keyword pre-filter for
unambiguous cases (an exact SKU pattern strongly signals a cost question; "summarize"/"update me on"
signals summarization) that skips an LLM call entirely when confident, falling back to LLM-based
intent classification (with structured output so the result is a typed literal) for ambiguous
messages. Model this as a LangGraph `StateGraph`: a `route` node sets `intent` in shared state, a
conditional edge dispatches to one of five feature nodes (project RAG, cost RAG, summarize, extract
demand, recommend), and each node wraps that chapter's LCEL chain. Handle the case where a query
legitimately needs more than one sub-system (e.g., "what's the status of X, and what would something
similar cost") with parallel branch execution and response merging, rather than forcing a single
intent label. Mention checkpointed state as the mechanism that lets a multi-turn interaction (e.g.,
entity extraction realizing a field is missing and asking a follow-up) resume correctly across
separate HTTP requests.

### Q15. Design the data model and access pattern for keeping five different clients' project data completely isolated in the same platform.

**A.** Combine Chapter 2's Pinecone multi-tenancy design with an application-layer authorization check:
Pinecone namespace = client_id as the hard boundary (never client-selectable — derived server-side
from the authenticated session), metadata filters (`project_id`, `doc_type`) for relevance within
that namespace, and the same `client_id` propagated through every downstream sub-system's state
(LangGraph's shared state object, Chapter 7) so summarization, entity extraction, and recommendation
queries are scoped identically to retrieval queries — a single point of truth for "which client is
this" rather than each sub-system independently trusting client-supplied identifiers. For defense in
depth, log every retrieval call's resolved namespace/filter alongside the response, so a data-leak
incident (if it ever happened) would be auditable after the fact.

### Q16. The platform needs to add a sixth capability (say, contract-status lookups) six months from now. How does your architecture accommodate that without a rewrite?

**A.** This is the payoff of the LangGraph design (Chapter 7) over a hand-rolled if/elif router: adding
a capability is "add a node implementing the new LCEL chain, add a conditional edge from the router,
extend the router's intent classification to include the new label" — no change to the other five
nodes, and the shared state schema (`client_id`, translated message, intent, response) likely doesn't
need to change either. If the new capability needs its own retrieval index, it's a new Pinecone
namespace/index following the exact same pattern as project data and the cost catalog (Chapter 2),
not a new storage paradigm. Name the actual limiting factor honestly: the router's intent
classification accuracy degrades somewhat as the label set grows, so periodically revisiting router
prompt/few-shot examples (or moving to a small fine-tuned classifier if the label set gets large) is
the real maintenance cost, not the graph structure itself.

## "What Would You Change If You Rebuilt This Today?"

### Q17. Knowing what's changed in the GenAI ecosystem since this was built, what would you change if you rebuilt this today?

**A.** A few concrete, defensible answers, worth having two or three ready rather than one generic one:

- **LangGraph maturity**: at the time this class of platform was likely first built, LangGraph was
  new or LangGraph-shaped orchestration was hand-rolled with nested chains/conditionals. Today,
  LangGraph's checkpointing, built-in human-in-the-loop interrupts, and native multi-agent patterns
  would replace a good amount of custom routing/state-management code (Chapter 7) — less bespoke
  infrastructure to maintain.
- **Multilingual embeddings**: rather than translate-then-embed (Chapter 7's translation layer sitting
  before retrieval), today's multilingual-native embedding models are strong enough to embed source
  documents and queries directly in their original languages into a shared space, removing a
  potentially lossy pre-retrieval translation step and its failure mode (Q5).
  - Trade-off to acknowledge: this only pays off if the multilingual model's quality on the *specific*
    language pairs in play is verified to be strong — otherwise translate-then-embed with a
    well-understood English-centric embedding model can still be the safer choice.
- **Hybrid search as a native vector-DB feature**: sparse+dense hybrid search (Chapter 3) has moved
  from "roll your own fusion in the application layer" toward native support in several vector
  databases — reducing the need to maintain a separate BM25 index and RRF logic by hand.
- **Structured output reliability**: function-calling/structured-output APIs (Chapter 5) have become
  materially more reliable since early GPT-3.5-era function calling, meaning less defensive
  validation code is needed around LLM-based entity extraction than would have been prudent at the
  time.
- **Evaluation tooling**: RAG-specific evaluation frameworks (e.g., Ragas — covered in course
  02-Capco-Model-Risk-Monitoring) matured after this kind of platform would first have shipped;
  today, building in faithfulness/relevance evaluation from day one, rather than bolting it on later,
  would be the default rather than an afterthought.

Close this answer by naming the meta-point: the five-features-in-one-platform shape was the right
call and wouldn't change — what would change is which off-the-shelf pieces you'd now trust instead of
building yourself.

## Client & Production Deployment

This platform served two real, competing pharma clients — **Eli Lilly** and **AstraZeneca** — live in
production on one shared orchestration codebase, which is exactly the kind of detail an interviewer
digs into once they realize this wasn't a single-tenant internal tool. The four questions below are
the ones most likely to follow once that comes up.

### Q18. This platform served two pharma clients, Eli Lilly and AstraZeneca, live in production — how did you guarantee one client's project data, cost catalog, or conversation history could never leak into the other's session?

**A.** Layered isolation, enforced as low in the stack as possible rather than relying on application
code to always get it right (Chapter 2, Chapter 00's production deployment diagram):

- **Pinecone namespace per client** (`eli-lilly` / `astrazeneca`) is the hard boundary for all three
  RAG-backed features — project data, cost catalog, and the embedding-similarity recommenders. A
  query against one namespace is physically incapable of returning another namespace's vectors; this
  isn't a `WHERE client_id = ...` filter that a missed line of code could bypass.
- **S3 prefixes/buckets per client, with IAM policies scoped accordingly**, for the documents feeding
  RAG — Eli Lilly's source documents are never in a location AstraZeneca's IAM role can read, and vice
  versa.
- **Conversation history/memory scoped by the same `client_id`** propagated through LangGraph's shared
  state object (Chapter 7) — every node in the graph (router, RAG, summarization, extraction,
  recommender) reads that one resolved `client_id`, rather than each sub-system independently trusting
  a client-supplied identifier, so there's a single point of truth for "whose session is this" instead
  of five places that each need to get isolation right.
- **The `client_id`/namespace value is always resolved server-side from the authenticated session** —
  never taken from a request parameter — which is the detail that actually prevents a compromised or
  malicious client input from selecting another client's namespace.
- **Defense in depth**: every retrieval call's resolved namespace and filter gets logged alongside the
  response, so a leak — if one ever happened — would be auditable after the fact, not invisible.

The honest framing for an interviewer: this isn't one clever trick, it's the same isolation boundary
(client identity, resolved server-side) enforced consistently at every layer — vector store, document
store, and conversation state — because a leak between two competing pharma companies' data is a
client-trust-ending incident, not a bug to file and fix later.

### Q19. Why ECS Fargate instead of pure Lambda for the orchestration layer here, given this is a live chat/voice interface?

**A.** See Chapter 7's "Running the Orchestration Layer on ECS Fargate in Production" section. Three
reasons, in order of how often they come up: (1) a LangGraph session is **stateful across turns**, not
a single stateless request — a warm, long-running process can hold connections to Pinecone, the LLM
API, and a state-store/checkpointer across a multi-turn conversation, where Lambda's per-invocation
model would mean reconnecting cold on every turn; (2) **cold starts are a latency problem specifically
because this is a live voice interface** — a delay that's tolerable on an async webhook reads as a
broken assistant on a phone call, and an always-warm ECS service behind an ALB avoids that; (3)
**checkpointed state needs somewhere to live between HTTP requests**, and a persistent service
maintaining a connection pool to that state store is a more natural fit than a fleet of independently
cold-starting functions each reconnecting from scratch. Lambda still earns its place elsewhere in the
same platform — voice-callback webhooks via Lambda + API Gateway are exactly the short-lived,
event-driven, bursty workload Lambda is built for. The rule of thumb: Lambda for event-driven/
stateless/bursty, ECS Fargate for long-running/stateful/latency-sensitive — this platform uses both,
deliberately, rather than treating "serverless" as a blanket policy.

### Q20. How do you keep p95 latency acceptable for a live voice interaction when the pipeline involves RAG retrieval, an LLM call, and possibly a recommender step?

**A.** See Chapter 7's "Latency Budget Management for a Live Voice Interface" section. The levers, in
rough order of impact: **streaming** — LangServe's `/stream` endpoint lets LLM generation emit tokens
as they're produced, paired with streaming text-to-speech so the client starts hearing a response
before generation finishes, which is the single biggest perceived-latency win; a **cheap rule/keyword
pre-filter ahead of LLM-based intent routing** (Chapter 7's router design) so unambiguous queries skip
an LLM round trip entirely; **warm infrastructure end to end** — the direct payoff of ECS Fargate over
Lambda (Q19), no cold start stacked on top of an already latency-sensitive pipeline; and **per-client
p95/p99 monitoring in CloudWatch**, not just an aggregate number, since two clients sharing one
orchestration fleet means one client's heavier query mix or larger project histories can quietly
degrade their own latency while the aggregate still looks healthy. Underneath all of that, the same
architectural choices from earlier chapters still matter for raw latency — cropping/chunking well so
retrieval doesn't need to scan huge contexts (Chapter 1), and being deliberate about when re-ranking
or hybrid search's extra pass is actually worth its added latency (Chapters 1 and 3) versus when plain
dense retrieval is fast enough.

### Q21. How would you roll out a change to the LangGraph orchestration logic to one client without risking the other client's live sessions?

**A.** This is where the shared-codebase, per-client-isolated-data design (Q18) has to extend to
deployment, not just data. A few concrete mechanisms, combinable: (1) **feature-flag or config-driven
branching keyed on `client_id`** inside the graph itself — a new node or changed routing behavior is
gated behind a flag scoped to one client's namespace, so the same deployed container serves both
clients but only one of them exercises the new code path; (2) **canary/rolling deploy at the ECS
service level with fast rollback** — deploy the new task definition to a small percentage of ECS
tasks behind the ALB first, watch per-client CloudWatch metrics (error rate, p95 latency, namespace-
scoped) before shifting the rest of the traffic, and roll back immediately if the canary's metrics for
either client regress; (3) if the change is substantial enough to be risky, **standing up a
parallel environment or a separate ECS service/task-set for the client getting the change**, so a
regression is contained to that client's traffic path entirely rather than shared infrastructure — a
heavier option, reserved for genuinely risky changes rather than routine iteration. In all cases, the
underlying enabler is the same one from Q18: because `client_id` is already the isolation key running
through the data layer (Pinecone namespace, S3 prefix), it's a natural key to also gate code paths and
rollout stages on, rather than needing a second, unrelated mechanism for deployment safety.
