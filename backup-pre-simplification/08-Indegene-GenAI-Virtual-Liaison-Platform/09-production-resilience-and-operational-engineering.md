# 09 — Production Resilience and Operational Engineering

This chapter matches course 5's chapter 06 in shape and intent: not architecture diagrams, but the
concrete, sometimes-unglamorous detail of how this platform actually behaves under failure and load —
a realistic error-handling table for the RAG pipeline, a scaling caveat specific to a live chat/voice
interface running on ECS Fargate, four "bug found and fixed" narratives specific to a RAG/LangGraph
platform, concrete timeout and retry values, and one candidly-named hardening gap.

**The same honesty note as chapter 08 applies throughout**: this course has no real source repository
behind it. Everything below is a plausible, technically detailed illustration of how a platform like
this would realistically be operated and where it would realistically break — written with real
specificity so it's useful to reason from and defend in an interview, not presented as a verified fact
about Indegene's actual implementation.

## A realistic error-handling table for the RAG pipeline

The organizing principle worth stating up front, and the one an interviewer is really listening for:
**retrieval-layer failures should degrade gracefully into an honest "I don't have enough information"
answer; generation-layer failures should fail loudly rather than leave a client waiting on dead air.**
Those are different failure philosophies for different reasons — a retrieval miss still has a safe,
honest fallback (say so), while a generation failure means there's no response at all to give, and
silence on a live voice call is worse than an explicit "I'm having trouble right now."

| Failure | What should happen |
|---|---|
| Pinecone query times out | Retry once with a short backoff (see timeout table below); if the retry also fails, degrade to "I don't have enough information to answer that right now" rather than propagating a raw exception to the chat/voice UI |
| Pinecone query succeeds but returns no matches above a minimum relevance threshold | Treated identically to a retrieval failure — explicitly tell the client no grounding was found, rather than letting the LLM generate an answer from parametric knowledge alone and presenting it as if it were grounded |
| The query-embedding call itself fails (embedding API error, not Pinecone) | Cannot search at all without a query vector — degrade immediately, don't attempt a Pinecone call with a stale or default vector |
| Namespace resolution fails (a session/auth problem prevents deriving `client_id` server-side) | **Hard fail** — return an explicit 401/403 to the client immediately; never silently fall back to a default or empty namespace (this is the same failure mode as the namespace-scoping bug below, and it should never resolve itself by accident) |
| LLM generation call times out or errors (Sagemaker/Bedrock endpoint issue) | Retry once with backoff on a transient (5xx/timeout) error only; on a second failure, surface an explicit "I'm having trouble generating a response, please try again" rather than a silent hang — especially important on a live voice call, where silence reads as a dropped or frozen assistant |
| Entity-extraction structured-output call returns malformed/unparseable JSON | One retry with a stricter "return valid JSON matching this schema only" reminder appended; on a second failure, fall back to a clarifying follow-up question to the client rather than guessing field values |
| Summarization map-reduce: one chunk's map-step summarization call fails | The other chunks' summaries still proceed independently (isolated per-chunk error handling); the reduce step explicitly checks whether it received a summary for every chunk it sent, and if not, prefaces the final summary as partial/incomplete rather than silently presenting a confident-sounding summary built from fewer inputs than it should have had |
| Speech-to-text returns empty or low-confidence output | The assistant asks the client to repeat, rather than passing an empty or garbled string into the LangGraph router and getting a nonsensical routed response |
| Translation-layer call fails (pre-retrieval, for a non-English client message) | Hard fail with a clear "having trouble understanding, could you try again" — proceeding with an untranslated message into English-tuned retrieval/routing would silently produce poor results rather than an obvious error |

Two patterns worth naming explicitly if asked "how does this pipeline handle failure":

- **Retrieval-adjacent failures degrade to an honest non-answer, never to an ungrounded guess.** The
  single most important discipline in this table is that a Pinecone timeout, a zero-match retrieval, or
  a failed embedding call should never result in the LLM quietly answering from its own parametric
  knowledge as if it were grounded — that's a worse outcome than telling the client the system doesn't
  know, especially for cost and status questions where an ungrounded-but-plausible-sounding answer is
  actively misleading.
- **Generation-layer and input-layer failures (LLM errors, bad transcription, bad translation) fail
  visibly.** Unlike a retrieval miss, there's no safe partial answer to fall back to — the honest
  response is an explicit "something went wrong," surfaced promptly, especially on a voice channel where
  a client has no way to distinguish "the system is thinking" from "the system silently died."

## The concurrency/scaling caveat: an in-process cache the LangGraph checkpointer doesn't cover

Chapter 7 already covers the correct fix for **conversation state** — LangGraph's checkpointer persists
`LiaisonState` to an external store (Redis/DynamoDB) precisely so any ECS task behind the ALB can pick
up the next turn of a session, not just the task that handled the previous one. That part is solved by
design. The caveat worth naming here is narrower, and easy to miss precisely *because* the checkpointer
gives the impression that "session state" as a whole is already handled: **not everything a session
touches lives in the checkpointed state.**

A realistic implementation adds a small, per-process, in-memory cache alongside the checkpointer purely
for latency — for example, a short-TTL dict mapping recently-seen query text to its already-computed
embedding, so a client re-asking a near-identical follow-up within the same conversation doesn't re-pay
an embedding-API round trip:

```python
# A per-process latency optimization, deliberately NOT part of the checkpointed LiaisonState
_query_embedding_cache: dict[str, tuple[list[float], float]] = {}  # text -> (vector, expiry_ts)

def cached_embed(query_text: str, ttl_seconds: int = 300) -> list[float]:
    now = time.time()
    cached = _query_embedding_cache.get(query_text)
    if cached and cached[1] > now:
        return cached[0]
    vector = embed(query_text)
    _query_embedding_cache[query_text] = (vector, now + ttl_seconds)
    return vector
```

This is the same shape of gap as course 5's `memory_cache` bearer-token dict: a plain, per-process
Python dictionary that **does not survive a restart** and **is not shared across scaled-out ECS
tasks**. Concretely, if the ALB has no session affinity (the naive, simplest ALB configuration) and a
client's conversation turns land on different tasks, this cache almost never hits in practice — every
task sees each of that session's queries as new, re-computing the embedding every time. The critical
distinction worth stating plainly, because it's what keeps this a performance caveat rather than a
correctness one: **a cache miss here just means a normal, full-cost embedding call happens instead of a
cheap cached one — it never produces a wrong answer**, unlike the checkpointer, where a missed read
really would lose conversation continuity. This is a "what would you change at 10x scale" answer, not
an incident report.

**Fixes, if this became a real, measured latency problem under horizontal scaling** — the same two-option
framing course 5 uses for its equivalent caveat:

1. **A shared cache (Redis)** — the direct fix, replacing the per-process dict with one every ECS task
   can read and write, at the cost of a network hop per lookup and one more managed dependency.
2. **ALB sticky sessions** — if scale-out is common but a given client's session naturally clusters in a
   short time window, routing a session's requests consistently to the same task makes the in-process
   cache actually pay off without adding new infrastructure, at the cost of complicating rolling deploys
   (a draining task still holds sessions pinned to it) exactly as chapter 7 already notes for the
   broader session-affinity trade-off.

## Four bugs found and fixed, specific to a RAG/LangGraph platform

These are illustrative, plausible-for-this-platform bug narratives — the kind of concrete "tell me
about a bug you found" material this course was previously light on, matching course 5's four real
bug stories in spirit and specificity even though these are reconstructed rather than source-confirmed.

**1. A chunking bug that split the cost catalog mid-row.** The cost-catalog ingestion pipeline used a
fixed-size character splitter (Chapter 1's "fixed-size chunking," without the row-aware fallback
Chapter 1 recommends) on a catalog exported as a flat, semi-structured text table. A chunk boundary
landed between a SKU column and its price column on several rows, so one retrieved chunk contained
`SKU-LOC-JP-STD Japanese localization standard package` with no price, and the *next* chunk started
mid-row with a bare price and no SKU context. A client asking "what's the price for SKU-LOC-JP-STD"
sometimes got a retrieved chunk with the right SKU and the wrong (adjacent row's) price silently
stitched into the prompt, and the LLM generated a plausible-sounding but wrong figure. *What would have
caught this earlier*: a retrieval golden-set test asserting that for a fixed list of known SKUs, the
top retrieved chunk contains both the SKU token and a price on the same chunk — a structural assertion
about chunk boundaries, not just a check that retrieval returns *something* relevant-looking. Switching
to row-aware chunking (one chunk per catalog line item, never split mid-row, per Chapter 1's guidance
for structured data) closed the bug directly.

**2. A namespace-scoping bug caught before it reached live client traffic.** A newly added debug/admin
utility (a "show recent retrievals for this session" tool used during a feature rollout) called
`index.query(...)` through a helper function whose `namespace` parameter had a default value, and the
new tool's call site simply didn't pass one — relying on the default rather than explicitly resolving
`client_id` server-side, the pattern every other retrieval path in the codebase followed. In a shared
staging environment where both clients' namespaces had been seeded for QA testing, hitting that debug
tool without an explicit namespace argument returned whichever namespace the default resolved to,
which briefly meant one client's debug session could see the other client's seeded staging data. *What
would have caught this earlier*: making `namespace` a required keyword argument with no default at the
retriever-helper level (so a missing namespace is a `TypeError` at call time, not a silent fallback),
plus an automated namespace-isolation assertion test — exactly the kind built in
`notebooks/09_namespace_isolation_resilience_demo.ipynb` — run in CI against every retrieval code path,
not just the primary ones, so a new debug tool added later would have failed the same test immediately
rather than needing a human to notice in staging.

**3. An embedding-model version upgrade that silently made new vectors incompatible with old ones.**
The embedding model backing project-data and cost-catalog retrieval was upgraded to a newer version
with the same output dimensionality (so Pinecone's fixed-per-index `dimension` check, which would
loudly reject a dimension mismatch, had nothing to catch) but a materially different vector geometry.
Existing vectors embedded with the old model were never re-embedded; new queries, embedded with the new
model, were compared via cosine similarity against a mix of old- and new-model vectors sitting in the
same index. Because different embedding models produce vectors in unrelated geometric spaces, a
similarity score between a new-model query and an old-model vector isn't just "less accurate" — it's
numerically meaningless, though it looks like an ordinary, in-range score with nothing to flag it as
suspect. The result was quietly irrelevant retrieved chunks for a subset of queries, with no error
condition anywhere to alert on. *What would have caught this earlier*: a fixed golden-query regression
set, re-run and diffed for retrieval-quality drift before and after any embedding-model change — not
just a smoke test that the model loads and returns vectors of the right shape — plus tagging every
vector with an `embedding_model_version` metadata field (the same versioning instinct as Chapter 8) so a
mixed-model index is at least detectable and query-time filterable, and migrating via a dual-write,
full-re-embed-and-cutover window rather than swapping the model in place.

**4. A summarization chain that silently dropped the first chunk under concurrent load.** The map step
of the project-summary map-reduce chain (Chapter 4) launched one summarization call per chunk
concurrently and collected results into a list via repeated `.append()` calls from within each
coroutine's completion callback, rather than via `asyncio.gather(*tasks)`, which preserves input order
and guarantees a result for every task. Under high concurrent conversation load, a race in that
completion-callback bookkeeping occasionally meant the first chunk's summary — the one representing a
project's earliest history — was dropped before the reduce step ran, and the reduce step had no signal
that anything was missing, so it silently produced a summary omitting the original context for a
project's later revisions. *What would have caught this earlier*: replacing the manual
append-from-callback pattern with `asyncio.gather`, whose return value is guaranteed to be a list the
same length as the input; and a concurrency/load test specifically exercising the map-reduce path under
concurrent sessions (rather than only unit-testing it against a single chunk list at a time), paired
with a reduce-step sanity assertion — `len(partial_summaries) == len(chunks)` — that fails loudly
instead of quietly summarizing whatever happened to arrive.

The common thread across all four, worth stating as the takeaway: three of the four (chunking,
namespace-scoping, embedding-version) are bugs where **nothing throws an exception** — the system
returns a normal-looking, well-formed response that happens to be built on subtly wrong grounding.
That's a harder class of bug to catch than a crash, and it's exactly why golden-query regression
testing and structural retrieval assertions matter more for a RAG platform than they would for a
typical CRUD service — the failure mode here is confidently wrong, not visibly broken.

## Concrete timeout and retry values, and why they're set that way

- **Pinecone query (on the live request path, blocking a chat/voice turn): 2-second timeout, one retry
  with ~150ms backoff, then degrade** (per the error table above). This is deliberately aggressive —
  shorter than Pinecone's own typical p99 — because a query sits directly in a latency-sensitive path
  where Chapter 7 already establishes that seconds of delay read as a broken assistant on a live voice
  call; it's better to degrade to an honest non-answer quickly than to let a client wait out a slow
  retrieval that may not even come back.
- **Pinecone upsert (background sync path, Chapter 8's event-driven or nightly re-index workers, never
  on a live request path): 10-second timeout, up to 3 retries with exponential backoff.** This path has
  no client waiting on it, so it can afford to be patient and persistent rather than fast — the priority
  here is eventual consistency, not latency.
- **LLM generation call: an 8-second overall timeout for non-streaming calls, or a 3-second
  first-token budget when streaming (Chapter 7's `/stream` endpoint) — one retry on a transient 5xx or
  timeout, no retry on a 4xx** (a 4xx means the request itself was malformed, and retrying an
  unmodified malformed request just wastes the retry budget on a guaranteed second failure).
- **Connection pooling**: each ECS task maintains one reused HTTP client (a single `httpx`/`requests`
  session, not a new connection per call) to both Pinecone and the LLM endpoint, with a fixed pool size
  (illustratively, 20 connections) sized to the task's expected concurrent-turn count. The rationale
  mirrors course 5's `pool_recycle` setting for its SQL connection pool: without reuse, every one of a
  task's concurrent conversation turns pays a fresh TLS handshake to Pinecone and the LLM API, which is
  a meaningful, avoidable latency tax stacked directly onto an already latency-sensitive voice pipeline.

## The candidly-named hardening gap: API keys cached at task start, no live rotation path

Secrets Manager (per the root `README.md`'s AWS production context) holds the Pinecone and LLM API
keys, which is the right baseline. The honest gap worth naming plainly, rather than treating "it's in
Secrets Manager" as the end of the story: a realistic implementation reads those secrets **once, at ECS
task startup**, and caches them in-process for the task's lifetime — there's no code path that
re-reads Secrets Manager mid-flight and picks up a rotated value. If a key is rotated (a routine
security practice, or an incident response to a suspected leak), **already-running tasks keep using the
old, cached value** until they're recycled by a normal deploy or a scale-in/scale-out event — meaning a
rotation doesn't actually take effect fleet-wide until someone forces a task restart, which is easy to
forget as a required manual step after a rotation rather than something the rotation itself guarantees.
The fix is mechanical and low-risk: either poll Secrets Manager on an interval and swap the in-process
client's credentials without a restart, or — simpler and more standard — treat "force a rolling restart
of the ECS service" as a required, automated step of the rotation runbook itself (e.g., an EventBridge
rule reacting to a Secrets Manager rotation event by triggering an ECS service deployment), rather than
leaving it as a manual step someone has to remember.

## Two things worth doing right, and naming positively

- **Retrieval failures degrading to an honest "I don't have enough information" rather than an
  ungrounded guess** is a real, deliberate design choice, not just an error-handling default — it's the
  single most important thing standing between "the assistant is occasionally wrong" and "the assistant
  is occasionally *confidently* wrong," and it's worth citing directly if asked "what did you do to make
  this production-ready for a regulated life-sciences client."
- **Namespace resolution hard-failing rather than silently falling back** (the error table's second row
  from the bottom) is the same instinct — a missing or ambiguous piece of context should surface as a
  visible error immediately, not resolve itself to a default that happens to work most of the time and
  fails exactly when it matters most, which is precisely what the namespace-scoping bug above shows
  going wrong when that discipline lapses even briefly.

## Tying It Back

Production-grade for a RAG/LangGraph platform means the same thing it means for course 5's document
uploader service: the failure modes are known, bounded, and either gracefully degraded or loudly
surfaced — never silently wrong. This platform's actual profile, as reasoned through here: retrieval
failures degrade to an honest non-answer while generation failures fail visibly; one well-understood,
low-stakes scaling caveat (a per-process embedding cache that isn't shared across ECS tasks, whose
worst case is extra latency, never a wrong answer); four illustrative but specific bugs, three of which
share the harder-to-catch "no exception thrown, just quietly wrong grounding" shape that's particular to
retrieval-based systems; concrete, deliberately asymmetric timeout values reflecting which calls sit on
a live voice path and which don't; and one honestly-named hardening gap — cached credentials with no
live rotation path — that chapter 08's version-aware metadata pattern and this chapter's namespace and
retrieval discipline both model how to close: name the gap precisely, propose the mechanical fix, and
don't let "it's already in Secrets Manager" pass for "it's already solved."
