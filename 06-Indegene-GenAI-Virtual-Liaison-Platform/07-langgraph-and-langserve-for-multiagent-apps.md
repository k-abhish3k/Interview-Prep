# 07 — LangGraph & LangServe for Multi-Agent Apps

## Why this chapter matters

Every prior chapter built one sub-system in isolation: project RAG, cost-catalog RAG, summarization,
entity extraction, recommenders. But the resume bullet describes **one chat/voice platform** — a
single client message has to be routed to the right sub-system (or several), and the whole thing has
to be deployed as one service the chat/voice front end can call. That's exactly what LangGraph
(orchestration) and LangServe (deployment) are for, and both are named skills on the resume. This
chapter closes the course by tying the five features together into one coherent system — the
architecture diagram from `00-README.md`, made concrete in code.

## Why LCEL Chains Alone Aren't Enough Here

Chapter 1's LCEL chains (`prompt | llm | parser`) are great for a single linear pipeline, but this
platform needs something LCEL doesn't naturally provide: **conditional routing based on intent**, a
**shared, evolving state** across a multi-turn conversation (the client's language, their
authenticated `client_id`, an in-progress entity-extraction that's missing a field and needs a
follow-up question), and potentially **looping** (ask a clarifying question, wait for the reply,
re-attempt extraction). A straight-through pipe can't branch, loop, or hold onto conversation state
across turns in a structured way — which is precisely the gap LangGraph fills.

## LangGraph: Stateful, Graph-Based Orchestration

LangGraph models an application as a **graph of nodes**, where each node is a function (often wrapping
an LCEL chain) that reads and writes a shared **state** object, and **edges** — including conditional
edges — determine which node runs next.

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, END

class LiaisonState(TypedDict):
    client_id: str
    original_language: str
    message_en: str            # translated to a common working language
    intent: str                # set by the router node
    response: str

def route_intent(state: LiaisonState) -> LiaisonState:
    # LLM-based or lightweight classifier; see "Designing the Router" below
    state["intent"] = classify_intent(state["message_en"])
    return state

def project_rag_node(state: LiaisonState) -> LiaisonState:
    state["response"] = run_project_rag(state["client_id"], state["message_en"])
    return state

def cost_rag_node(state: LiaisonState) -> LiaisonState:
    state["response"] = run_cost_catalog_rag(state["message_en"])
    return state

def summarize_node(state: LiaisonState) -> LiaisonState:
    state["response"] = run_project_summary(state["client_id"], state["message_en"])
    return state

def extract_demand_node(state: LiaisonState) -> LiaisonState:
    state["response"] = run_entity_extraction(state["message_en"])
    return state

def recommend_node(state: LiaisonState) -> LiaisonState:
    state["response"] = run_recommender(state["client_id"], state["message_en"])
    return state

def choose_branch(state: LiaisonState) -> Literal[
    "project_rag", "cost_rag", "summarize", "extract_demand", "recommend"
]:
    return state["intent"]   # one of the five literal values, set by route_intent

graph = StateGraph(LiaisonState)
graph.add_node("route", route_intent)
graph.add_node("project_rag", project_rag_node)
graph.add_node("cost_rag", cost_rag_node)
graph.add_node("summarize", summarize_node)
graph.add_node("extract_demand", extract_demand_node)
graph.add_node("recommend", recommend_node)

graph.set_entry_point("route")
graph.add_conditional_edges("route", choose_branch, {
    "project_rag": "project_rag",
    "cost_rag": "cost_rag",
    "summarize": "summarize",
    "extract_demand": "extract_demand",
    "recommend": "recommend",
})
for node in ["project_rag", "cost_rag", "summarize", "extract_demand", "recommend"]:
    graph.add_edge(node, END)

app = graph.compile()
result = app.invoke({
    "client_id": "acme-pharma",
    "original_language": "ja",
    "message_en": "What's the status of the France localization for Project Atlas?",
})
```

This is the architecture diagram from `00-README.md`, expressed as executable graph structure: each of
the five feature chapters becomes one node, the router (`route_intent` / `choose_branch`) is the
"LangChain / LangGraph orchestration layer" box, and `END` funnels every branch back to a single
response.

### Designing the Router

The `route_intent` node deciding "does this need RAG retrieval, summarization, entity extraction, or
a recommendation" is itself worth designing deliberately — this is exactly the system-design question
"design the routing logic that decides which sub-system handles a query" from `99-Interview-QA.md`.
Two viable approaches, often combined:

- **LLM-based classification**: a small, cheap prompt that asks the model to classify the (already
  translated) message into one of the five intents, optionally with structured output (Chapter 5's
  technique, reused) so the result is a typed literal, not free text to parse.
- **Rule/keyword pre-filter**: cheap heuristics for unambiguous cases (a message containing an exact
  SKU pattern is very likely a cost question; "summarize" or "update me on" strongly signals the
  summarization intent) that skip the LLM call entirely for the obvious cases, falling back to LLM
  classification only for ambiguous messages — a latency and cost optimization worth mentioning.

A message can also legitimately need **more than one** node — a client might ask "what's the status
of Project Atlas, and what would a similar new project cost." LangGraph supports this with parallel
branches (`RunnableParallel`-style fan-out inside a node, or multiple conditional edges) that both
execute and then merge into a combined response, rather than forcing a single intent label per
message — worth raising if an interviewer pushes on "what if the query needs two of your five
sub-systems at once."

### Why a Graph Instead of Nested If/Else

You could write this routing as a chain of `if/elif` statements calling different LCEL chains, and for
a small number of static branches that's a legitimate simpler alternative — worth acknowledging rather
than overselling LangGraph as strictly necessary. LangGraph earns its complexity once you need:

- **Explicit, inspectable state** that persists across multiple turns of a conversation (e.g., the
  entity-extraction node realizes a required field is missing, asks a follow-up, and needs to resume
  extraction with the client's reply merged into the same state — a loop, not a straight line).
  LangGraph's state object and conditional edges (including edges back to an earlier node) model this
  naturally, where nested if/else in a single function does not.
  LangGraph's built-in **checkpointing** also means that in-progress state can be persisted and
  resumed across separate HTTP requests, which matters a great deal for a chat/voice platform where
  each turn may be a separate API call.
- **Observability**: LangGraph's execution can be traced node-by-node (e.g., via LangSmith), which
  turns "why did the assistant answer this way" into an inspectable graph trace instead of buried
  print statements in nested conditionals — valuable in a regulated life-sciences context where
  answers may need to be audited.
- **Extensibility**: adding a sixth sub-system later is "add a node and an edge," not "refactor a
  growing if/elif chain."

## LangServe: Deploying the Graph as a REST API

Once the graph (or an LCEL chain) is built, LangServe exposes it as a REST API with minimal glue code,
because everything in LangChain/LangGraph implements the same `Runnable` interface (Chapter 1's LCEL
chapter covers this same point for plain chains) — LangServe just needs a `Runnable`-shaped object to
wrap.

```python
from fastapi import FastAPI
from langserve import add_routes

api = FastAPI(title="Virtual Liaison API")
add_routes(api, app, path="/liaison")   # app = the compiled LangGraph from above

# uvicorn main:api --host 0.0.0.0 --port 8000
# POST /liaison/invoke  {"input": {"client_id": "...", "message_en": "..."}}
```

This single `add_routes` call gets the chat/voice front end an HTTP endpoint with `/invoke`,
`/batch`, and `/stream` variants for free — the streaming endpoint in particular matters for a chat
UI, where token-by-token streaming (Chapter 1's LCEL chapter point about perceived latency applies
here too) makes a multi-second RAG-plus-generation round trip feel responsive.

## Running the Orchestration Layer on ECS Fargate in Production

The LangServe app above (`api = FastAPI(...)`, `add_routes(...)`) is the code; in production for Eli
Lilly and AstraZeneca, that FastAPI app was containerized and run as a service on **AWS ECS
(Fargate)** behind an **ALB** — not deployed as a Lambda function per request. Worth being able to
justify that choice directly, since "why not just Lambda, it's serverless and cheaper" is a fair
question for a chat/voice interface.

### Why ECS Fargate Over Pure Lambda Here

- **A LangGraph session is stateful across turns, not a single stateless request.** A voice/chat
  conversation may span several turns (the entity-extraction node above asking a follow-up question
  and waiting for the reply is a real example), and each turn benefits from a warm process holding
  connections (to Pinecone, to the LLM API, to the translation service) rather than re-establishing
  them cold on every invocation. Lambda's execution model is built around short-lived, largely
  stateless invocations; a long-running ECS service keeps those connections and any in-memory caches
  warm across the whole session.
- **Cold starts are a latency problem for a live voice interface in a way they aren't for an async
  webhook.** A Lambda cold start (hundreds of milliseconds to a few seconds, depending on package
  size and runtime) is often tolerable for a background job; it reads as a broken, hanging assistant
  on a live voice call where the client is waiting for a spoken response. An ECS Fargate service
  behind an ALB stays warm and serves requests with consistent latency instead.
- **LangGraph's checkpointing (Chapter 7's earlier section) needs a place to persist state between
  HTTP requests.** ECS services can maintain a connection pool to a shared state store (e.g., a
  managed Redis/DynamoDB-backed checkpointer) far more naturally than a fleet of independently
  cold-starting Lambda invocations, each of which would need to reconnect to that state store from
  scratch.
- **Lambda still earns its place elsewhere in the same platform** — the architecture (see
  `00-README.md`'s production deployment diagram) uses **Lambda + API Gateway** specifically for
  lightweight, event-driven endpoints like voice-callback webhooks, where a short-lived, bursty,
  stateless invocation is exactly the right fit. The rule of thumb worth stating in an interview: pick
  Lambda for event-driven, stateless, bursty workloads, and a container service (ECS Fargate) for a
  long-running, stateful, latency-sensitive orchestration layer — this platform uses both,
  deliberately, for different parts of the system rather than treating "serverless" as a blanket
  policy.

### ALB and Session/State Considerations

Because the orchestration layer runs as a fleet of ECS tasks behind an ALB rather than a single
process, a given conversation's turns can, in principle, land on different tasks. Two ways to handle
that, and the second is the one that scales:

- **ALB session affinity (sticky sessions)** routes all of one client's requests to the same target
  for the life of a session — simple, but it couples session continuity to a specific task staying
  healthy, and complicates rolling deploys/autoscaling (a task that needs to drain still has
  in-flight sessions pinned to it).
- **External state store, no server-side affinity required** — LangGraph's checkpointer persists
  conversation state (the `LiaisonState` object) to a shared store (Redis, DynamoDB) keyed by session
  ID, so **any** ECS task can pick up the next turn of a conversation by loading that state, not just
  the task that handled the previous turn. This is the more production-robust pattern: it decouples
  session continuity from any individual task's lifecycle, which matters a lot for rolling deploys and
  autoscaling under real daily multilingual client traffic from two separate client organizations.

### Latency Budget Management for a Live Voice Interface

A live voice turn on this platform involves several sequential-feeling steps — speech-to-text,
language detection/translation, intent routing, RAG retrieval (possibly hybrid search), LLM
generation, translation back, text-to-speech — and a client waiting on the line notices seconds, not
just tail latency. A few concrete levers, on top of the ECS-vs-Lambda choice above:

- **Streaming wherever the modality allows it**: LangServe's `/stream` endpoint (covered above) lets
  the LLM generation step emit tokens as they're produced rather than waiting for the full response,
  and pairing that with streaming text-to-speech means the client starts hearing a response before
  generation has finished — the single biggest perceived-latency win available here.
- **Keeping the router's fast path cheap**: the rule/keyword pre-filter ahead of LLM-based intent
  classification (the "Designing the Router" section above) exists as much for latency as for cost —
  skipping an LLM round trip for unambiguous cases matters more on a voice call than in a chat window
  where a user tolerates a bit more delay.
- **Warm infrastructure, end to end**: this is the direct payoff of ECS Fargate over Lambda for this
  service — no cold start added on top of an already latency-sensitive pipeline.
- **Per-client latency monitoring in CloudWatch**: with two clients sharing the same orchestration
  fleet, tracking p95/p99 latency **per client namespace**, not just an aggregate, matters — an
  aggregate number can look healthy while one client's traffic (larger project histories, a heavier
  query mix) is quietly degrading, and per-client dashboards catch that before an interviewer would
  even think to ask about it.

## Where Voice and Translation Fit

The voice and multilingual pieces of the platform sit **outside** the graph proper, as pre/post
processing: speech-to-text and language detection happen before the message enters the graph
(normalizing to a common working language so every node — router, RAG, extraction — operates on
consistent input, rather than every node needing to handle N languages independently), and
text-to-speech plus translation-back-to-client-language happen after the graph produces its response.
That separation — "the graph handles intent/orchestration; a thin pre/post layer handles modality and
language" — is a clean way to describe the architecture without conflating two genuinely different
concerns.

## Tying It Back

When asked "how did you tie the five features into one platform," the answer is a LangGraph
`StateGraph` with a router node that classifies intent (LLM-based, with a rule-based fast path for
unambiguous cases) and conditionally dispatches to one of five nodes — project RAG, cost-catalog RAG,
summarization, entity extraction, recommendation — each wrapping the LCEL chain from its respective
chapter, sharing one state object that carries the authenticated client context across a multi-turn
conversation. The compiled graph is deployed via LangServe as a REST API with streaming support,
called by a chat/voice front end that handles language detection/translation and speech-to-text/
text-to-speech outside the graph, on either side of it. This is also the natural jumping-off point for
"what would you change if you rebuilt this today" in `99-Interview-QA.md` — LangGraph itself matured
substantially after this kind of platform would first have been built, and an early version likely
used more ad hoc branching logic than a formal graph.
