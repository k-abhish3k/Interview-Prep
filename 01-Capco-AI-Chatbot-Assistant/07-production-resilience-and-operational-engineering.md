# 07 — Production Resilience and Operational Engineering

## Why this chapter exists

Chapter 3 covers what makes this chatbot *answer well* and what makes it *safe to run at a bank* from
an architecture-diagram level — WAF, private networking, Azure AD, deployment slots. This chapter is
the layer underneath that: the realistic error-handling behavior a system like this actually has, the
concrete scaling caveat specific to running a stateful-feeling chatbot across a stateless, horizontally
scaled fleet, the kind of GenAI-specific bugs that are genuinely easy to ship and painful to find, exact
timeout/retry parameters worth having memorized, and one hardening gap named candidly rather than
glossed over.

**The same honesty constraint applies here as in Chapter 6**: this is an illustrative, plausible
reconstruction of what a production-grade Azure OpenAI chatbot's operational engineering looks like —
not a verified description of a specific system's source code, the way course 05 was rebuilt from real
proprietary code. Read it as a technically confident, defensible system-design and debugging answer,
not as a factual incident log.

## A realistic error-handling table

What a system like this would actually need to do on each failure — and, as importantly, which
failures should be silently absorbed versus surfaced to the user versus paged to an on-call engineer:

| Failure | What happens | Why |
|---|---|---|
| Azure OpenAI returns `429` (rate-limited) | Retried with exponential backoff (see below); if retries exhaust, user sees "I'm getting a lot of requests right now — please try again in a moment," not a raw error | `429` is transient and expected under real load; the user-facing message should never look like a bug |
| Azure OpenAI content filter blocks a prompt or completion | `400`-class response from the API is caught explicitly and mapped to a scoped, generic "I can't help with that request" reply — never the raw filter category/reason surfaced to the user | Surfacing the filter's internal category taxonomy to an end user is both unhelpful and a minor information-disclosure smell; the *fact* it was filtered is logged with full detail for review |
| Azure AI Search retrieval call fails or times out | The chain falls back to answering from the system prompt/general instructions alone, with the response explicitly caveated ("I couldn't check our documentation just now, so treat this as a general answer") rather than failing the whole turn | A retrieval outage shouldn't take down the entire chatbot when the model can still say *something* honest and useful |
| Azure AI Search returns zero relevant chunks | Not an error at all — the model is instructed (Chapter 1) to say it doesn't know, rather than answering ungrounded | This is a *correct* outcome, not a failure, and shouldn't be logged or alerted as one |
| Session/conversation store (Redis/Cosmos/SQL) unreachable | `503`-class response, user sees "temporarily unavailable, please retry" — the request is **not** silently served with an empty history, because that would let the model answer as if this were a brand-new conversation without saying so | Silently dropping conversation state is a worse user experience than an honest, brief outage message |
| A single message in a batch/background evaluation job fails | That item is recorded as failed and the batch continues; the whole batch is not aborted for one bad input | Mirrors the general "isolate one failure from the rest of the batch" pattern that shows up across this candidate's other production services |
| Streaming connection drops mid-response (client disconnect, proxy timeout) | The partial response already generated is still persisted to conversation history server-side (so the model's own next turn has continuity), but nothing further is sent to a client that's gone | Avoids wasting a completed generation and avoids the model "forgetting" its own prior answer on the next turn just because the UI didn't render it fully |
| Azure OpenAI SDK call raises an unexpected exception (not a mapped 429/400/5xx) | Logged with full stack trace and request ID to Application Insights; user sees a generic "something went wrong" message; **not** retried automatically, since an unrecognized failure mode is exactly the case where blind retry risk (Bug 3, below) is highest | Unknown failures should fail loud to the engineering team and fail soft to the user — never both loud to the user and silent to the team |

Two patterns worth naming explicitly if asked "how does this chatbot handle failure, as a design
philosophy":

- **User-visible messages are deliberately generic and reassuring; log entries are deliberately
  detailed.** The user never sees a stack trace, an HTTP status code, or a content-filter category —
  they see one of a small, curated set of plain-language fallback messages. Everything an engineer would
  need to actually diagnose the failure — request ID, which stage failed (retrieval vs. generation vs.
  session store), latency at each stage, model/deployment version — goes to Application Insights, tied
  to the same per-conversation trace described in Chapter 3.
- **A retrieval failure degrades the answer; almost nothing else does.** The one deliberate exception
  to "fail closed" is retrieval — because the alternative (refusing to answer at all whenever Azure AI
  Search hiccups) turns a partial, recoverable degradation into a full outage of the entire chatbot for
  a dependency that isn't strictly required to produce *some* answer.

## The scaling/concurrency caveat: per-instance state under a stateless-looking fleet

Chapter 3 states the App Service compute layer is stateless and scales out horizontally without sticky
sessions — that's true of the *conversation history*, which lives in an external session store. It is
**not** true of everything else a chatbot process plausibly holds in memory, and this is the concrete,
specific caveat worth naming rather than letting "stateless" stand as an unqualified claim:

- **An in-process full-answer cache** (Chapter 6) — if implemented as a plain in-memory dict rather
  than a shared Redis instance, each App Service instance builds up its own independent cache. Two
  concrete consequences: cache **hit rate fragments** across instances (a question answered and cached
  on instance A is a cold miss on instance B, even seconds later), and cache **invalidation doesn't
  propagate** — the event-driven purge from Chapter 6's proposed design has to reach every instance
  individually, or a stale answer can keep being served from whichever instance the load balancer
  happens to route a given user to, even after the "correct" instances have already purged it.
- **An in-process system-prompt/config cache**, loaded once at process startup from a config store, has
  the same shape of problem in a different flavor: **prompt/config version skew during a rolling
  deployment.** While a new build is being rolled out slot-by-slot or instance-by-instance, some
  instances are still running the old process (old cached prompt version) and some are running the new
  one, simultaneously, in production, for however long the rollout window is. A user's conversation
  could plausibly get answered under two different system-prompt versions on two consecutive turns if
  load-balanced without session affinity across that window — not a data-corruption risk, but a real,
  observable behavior inconsistency that's easy to misdiagnose as "the model behaving inconsistently"
  when the actual cause is fleet-level config skew, not the model.
- **What doesn't have this problem, and why it's worth contrasting:** the conversation-history session
  store and (per Chapter 6's proposal) the retrieval index and embedding cache are all externalized —
  Redis, Cosmos/SQL, Azure AI Search — reachable identically from every instance, so a request landing
  on any instance sees the same conversation state and the same retrieval results. The caveat is
  specifically, narrowly about whatever's still sitting in a plain Python dict inside a single process,
  the same class of gap course 05's own maintenance-loop and token-cache caveats describe for a
  different service — it's a recurring shape of bug across stateless-fleet architectures, not a
  one-off.

**How you'd harden this, if either became a real reported problem** — worth naming as concrete options
rather than a vague "add caching":

1. **Move both caches to a shared store (Redis / Azure Cache for Redis).** Direct fix for the answer
   cache; for the config cache, pair it with a short TTL (e.g. re-read every 60 seconds) or a
   push-based invalidation (a lightweight pub/sub notification on config change) rather than
   process-lifetime caching, so a compliance-driven prompt correction propagates within seconds, not
   only on the next full restart.
2. **Session affinity at the load balancer**, if a shared store isn't justified yet — routing a given
   user's requests consistently to the same instance sidesteps the fragmentation problem without adding
   a new managed dependency, at the cost of losing even load distribution and needing every instance to
   independently recover its own state on a scale-in event.

## Four bugs specific to a GenAI chatbot, and what would have caught each earlier

Illustrative, plausible bug narratives — the kind of concrete, named failure a candid engineer can
describe from a project like this, not a claim about a specific verified incident.

**1. A truncation off-by-one that silently dropped the system prompt, not just old turns.** The
sliding-window truncation logic (Chapter 3's memory strategy, implemented in
`notebooks/03_simple_chatbot_with_memory.ipynb`) is supposed to always keep the system prompt and drop
*only* old conversation turns once the token budget is exceeded. A later refactor that moved token
accounting to compute the system prompt's cost *inside* the same backward-walking loop as the
conversation turns — instead of reserving its budget up front — meant that on any conversation long
enough to fill the budget exactly at the system prompt's own boundary, the loop's `if cost > budget:
break` could trigger before the system prompt had been counted at all, silently evicting it along with
the oldest turns. The visible symptom was subtle and easy to misattribute: in long conversations only,
the bot would occasionally answer questions it should have refused (general financial advice, questions
outside its documented scope) — because the very instructions telling it not to were the thing missing
from that call's prompt, and nothing errored, since a request with no system message is still a
perfectly valid Azure OpenAI call. *What would have caught it earlier:* a unit test asserting the
system prompt is present in the output of the truncation function for *every* history length, including
lengths chosen specifically to sit right at the budget boundary — not just a "short history" and "long
history" happy-path pair.

**2. A streaming loop that dropped the final token under a specific finish-reason ordering.** The
backend's SSE streaming endpoint, consumed by the `react-service` frontend (Chapter 4), iterated Azure OpenAI's streamed chunks and checked
`chunk.choices[0].finish_reason == "stop"` to know when to stop the loop and call `msg.update()` —
written under the (usually true) assumption that the terminal chunk carries `finish_reason="stop"` with
an empty `delta.content`. In a minority of completions, the terminal chunk instead carried *both* the
last piece of visible content **and** `finish_reason="stop"` in the same server-sent event. Because the
loop checked `finish_reason` and broke before appending that chunk's `delta.content`, the last word (or,
worse, the closing citation label) of a small fraction of answers was silently dropped — from what the
user saw on screen, and from what got written into conversation history, meaning every *subsequent* turn's
context was missing that last word too, since the persisted assistant turn was the truncated one. *What
would have caught it earlier:* a "streaming equivalence" integration test — for a fixed prompt/seed,
assert that concatenating every chunk from `.stream(...)` produces the exact same text as a single
non-streaming `.invoke(...)` call, run across enough sample completions to catch a low-frequency
ordering issue rather than relying on eyeballing a handful of manual tests.

**3. A retry wrapper that double-called Azure OpenAI on a client-side timeout mid-stream.** A
`tenacity`-based retry decorator (`stop_after_attempt(3)`, exponential backoff) wrapped the entire
"retrieve context, then call Azure OpenAI" function as a single unit, on the reasoning that any failure
in that function was safe to simply retry from scratch. That reasoning holds for a failure *before* the
first byte of the response comes back — a connection refused, a DNS failure, a `429` — but not for a
client-side read timeout that fires *after* the server has already started streaming a long completion:
the HTTP request had already reached Azure OpenAI and generation (and billing) was already underway when
the client gave up waiting and the decorator fired a second, fully independent call. Occasionally, this
produced a visibly broken UI where two responses' tokens interleaved into the same message as both
streams happened to still be in flight briefly overlapping. The root cause was retrying at too coarse a
granularity — wrapping the whole "already-connected, mid-stream" case the same as the "never even
connected" case — rather than distinguishing them. *What would have caught it earlier:* a test against a
mock endpoint that simulates a slow/hanging response *after* emitting a few real chunks, asserting
exactly one upstream call is made and exactly one message is ever written to the UI/history for that
turn — plus a code-review rule of thumb worth stating explicitly: only retry a call automatically if you
can prove it never got a response started, not just "it didn't finish quickly."

**4. A full-answer cache serving a stale answer for hours after the source document was corrected.**
The in-process answer cache (Chapter 6) keyed on normalized question text with a multi-hour TTL and no
event-driven invalidation meant that a compliance-driven correction to a policy answer — say, a refund
window changing from 5–7 to 3–5 business days, pushed and reflected in the retrieval index within
minutes — could still be served verbatim from cache for the remainder of its TTL window, even though a
brand-new question asking the same thing (with no cache hit) would have gotten the corrected answer
immediately. This was only caught when a compliance reviewer spot-checking the bot's answers against the
just-corrected policy got the old number back. *What would have caught it earlier:* an automated
freshness regression test — ask a canonical question, simulate a document-update event for the source it
should be grounded in, ask the identical question again, and assert the answer actually changed —
exactly the gap Chapter 6's proposed event-driven cache invalidation is designed to close, and the
concrete reason its TTL recommendation for the answer cache specifically (10–15 minutes, not hours) is
deliberately short: a short TTL is the cheap fallback safety net for exactly this bug, in case the
event-driven purge itself ever fails to fire.

The common thread across all four, worth stating as the takeaway: **three of the four are "the code
runs without error, so it looks fine" bugs** — nothing throws an exception, nothing shows up in an error
rate dashboard, the response still renders — which is exactly the category of bug that only a test
asserting *correctness of content*, not just *absence of exceptions*, will catch. The retry bug is the
one exception, and it's the one most worth a deliberate code-review rule rather than a test, since the
double-call is a race that won't reproduce reliably on demand.

## Concrete timeout, retry, and pooling parameters for Azure OpenAI calls

Illustrative but specific — the kind of numbers worth having ready if asked "what timeout would you set
and why," rather than a vague "reasonable timeouts":

```python
# Illustrative parameter choices for an AzureOpenAI/httpx-based client
CONNECT_TIMEOUT_SECONDS = 5       # fail fast if Azure OpenAI's endpoint is unreachable at all
READ_TIMEOUT_SECONDS_STREAMING = 60   # generous -- a long completion streaming token-by-token
                                        # can legitimately take a while between chunks under load
READ_TIMEOUT_SECONDS_NON_STREAMING = 30

MAX_RETRIES = 3
BACKOFF_BASE_SECONDS = 2          # 2s, 4s, 8s before jitter
BACKOFF_MAX_SECONDS = 20          # cap the wait even after several attempts
JITTER = True                     # avoid every retrying client backing off in lockstep after a
                                    # shared outage ends -- a classic thundering-herd trigger

RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}
# 429 specifically: prefer the API's own Retry-After header over the computed backoff
# whenever Azure OpenAI supplies one -- it knows its own quota reset window better than a
# client-side guess does.

# Connection pooling (httpx, underlying the Azure OpenAI SDK's transport)
POOL_MAX_CONNECTIONS = 100
POOL_MAX_KEEPALIVE_CONNECTIONS = 20
```

Why these specific choices, beyond "reasonable defaults":

- **A short connect timeout (5s) paired with a much longer read timeout (30–60s)** reflects that these
  two failure modes mean very different things: a slow *connection* almost always means a network/DNS
  problem worth failing fast on and retrying, while a slow *response* on an already-established
  connection can simply mean a long completion at a busy moment — killing that too aggressively would
  abort perfectly healthy, if slow, generations.
- **Retrying only on the listed status codes, never on an unrecognized exception** (Bug 3 above) is a
  deliberate, narrow allowlist rather than a blanket "retry anything that raises" — the entire point is
  to only retry failures known to be safe and transient.
- **Respecting `Retry-After` over a computed backoff on `429`** matters because Azure OpenAI's own
  quota system knows the actual reset window; guessing with a fixed exponential schedule risks retrying
  either too early (wasting an attempt, extending the eventual wait) or too conservatively (leaving
  latency on the table the API itself said wasn't necessary).
- **Bounded connection pooling** matters at real concurrency — an unbounded pool under a traffic spike
  can open far more simultaneous connections to Azure OpenAI than the deployment's practical concurrency
  can usefully absorb, working against the TPM/RPM quota management story in Chapter 3 rather than
  helping it.

## A hardening gap, named candidly: a hardcoded fallback system prompt

A realistic version of this chatbot's config loading plausibly looks like: read the current system
prompt from a config store at process startup, cache it in-process for the life of that process (the
same in-process cache discussed in the scaling section above) — and, if that config-store call fails at
startup (a transient network blip, the config service being briefly unavailable during its own
deployment), fall back to a **hardcoded default system prompt string baked directly into source**, so
the process can still start and serve traffic rather than crash-looping on a dependency that's merely
having a bad minute.

That fallback is a reasonable resilience choice in isolation — a chatbot that refuses to start at all
because a config service hiccuped is arguably a worse outcome. The gap worth naming candidly is what
happens **next**: the hardcoded fallback is whatever was last written into source at the most recent
deploy, with no expiry, no alert, and no forced reconciliation once the config service recovers. If an
instance restarts (a scale-out event, a crash, a deploy) at exactly the wrong moment relative to the
config service's own availability, that instance can end up silently serving a **stale, possibly
several-versions-old system prompt indefinitely** — including, in the worst case, an outdated
scope-guarding or compliance instruction that a since-shipped correction was specifically meant to
replace — with nothing in the logs louder than a single startup-time warning that's easy to miss in a
noisy deploy.

The fix is straightforward and low-risk, the same shape as course 05's approach to its own named
hardening gap: keep the fallback (removing it trades one failure mode for a worse one), but make falling
back to it **loud and self-correcting** rather than silent and permanent — alert explicitly whenever a
process is running on the fallback prompt rather than the config-store value, and retry the config-store
read on a short interval (rather than only at startup) so an instance that started degraded
self-heals onto the current prompt version within minutes of the config service recovering, instead of
staying on stale instructions until its next restart.

## Two things worth citing as good practice, not just gaps

Not everything here is a caveat — two choices already described elsewhere in this course are genuinely
solid operational hygiene, worth having ready as a positive answer to "what did you do to make this
production-ready" rather than only having gaps to discuss:

- **Streaming time-to-first-token as the latency metric that actually matters**, tracked separately
  from total generation time (Chapter 3) — a concrete example of designing observability around what
  users actually perceive, not just what's easiest to measure.
- **Per-conversation Application Insights tracing** (Chapter 3) that already captures retrieved sources,
  model/deployment version, and content-filter outcome — exactly the data needed to diagnose three of
  the four bugs above after the fact (which retrieval result was used, which stage the latency came
  from, whether a content-filter event coincided with a user complaint), without needing to reproduce
  the failure live.

## Tying It Back

Production-grade for a GenAI chatbot isn't "the model gives good answers" — it's "every failure mode is
known, bounded, and has an honest answer for what the user sees versus what an engineer sees," and this
chapter's actual shape: a deliberately asymmetric error-handling table (degrade gracefully on retrieval,
fail loud on everything unrecognized), one concrete and specific scaling caveat (per-instance caches
under a stateless-looking fleet, with two named fixes), four GenAI-specific bugs that are exactly the
"looks fine, isn't" class that only correctness-focused tests catch, real timeout/retry numbers with the
reasoning behind each, and one candidly-named hardening gap (a hardcoded fallback prompt that can go
silently stale) with a proportionate, already-identified fix rather than either ignoring it or
over-engineering a solution to a problem that hasn't caused an incident yet.
