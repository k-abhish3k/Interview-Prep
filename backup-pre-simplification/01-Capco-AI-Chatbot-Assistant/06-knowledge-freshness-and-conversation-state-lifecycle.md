# 06 — Knowledge Freshness and Conversation-State Lifecycle

## Why this chapter exists

Course 05 (Document Uploader Service) has a chapter that exists because of a real interview question:
*"for revised versions of the same document, how are you handling those?"* That question doesn't stop
at the uploader — it has a direct sequel for anything that *answers questions from* those documents:
**"when the source documents behind this chatbot's answers get updated or corrected, how do you make
sure the bot stops answering from the old version?"** That's a natural, likely follow-up for a
RAG-grounded chatbot, and it deserves the same honest, structured treatment: what happens today, what
you can do about it right now with what exists, which two related-sounding concerns are easy to
conflate, a genuine proposed fix, and the downstream systems a real fix would have to reach.

**Everything below is an illustrative, plausible reconstruction**, exactly like the rest of this
course — there is no source repository backing this course the way course 05 was rebuilt from real
code. The goal is to be specific and technically confident about *what a system like this would do*,
not to claim any of these exact field names, numbers, or code paths are a verified description of a
real deployed system. Treat this chapter the way you'd treat a strong, well-reasoned system-design
answer in an interview: precise, defensible, and clearly labeled as reasoning rather than fact.

## Part 1 — What a realistic implementation does today: refresh-on-schedule, retire-on-nothing

A RAG chatbot like the one in Chapter 3 has two places where "the past" can leak into "the present":
the **retrieval index** (Azure AI Search) and any **cache** sitting between retrieval/generation and
the user. A realistic, typical implementation handles the first half of that — getting new content in
— reasonably well, and is much weaker on the second half — getting stale content *out*.

- **Index refresh is schedule-driven, not event-driven.** Azure AI Search's indexer runs on a
  recurring schedule (plausibly nightly, or on some fixed interval — say every 4 hours) against
  whatever Blob Storage container the source documents land in, which — per course 05's own scope
  note on its downstream retrieval layer — is plausibly fed by a service like the Document Uploader
  Service. Each run does a `mergeOrUpload` for whatever's new or changed at the blob level and chunks
  it into the index.

  Worth being precise about the mechanics if pressed on it: an Azure AI Search **indexer** is a
  distinct object binding a *data source* (the Blob container), a *target index* (your chunk schema),
  and a *schedule* (something like `PT4H`, an ISO-8601 interval — there's no sub-minute option, and
  critically, there's no built-in "fire on blob write" trigger). Getting closer to event-driven requires
  bolting on an Event Grid subscription that calls `Run Indexer` on demand whenever a blob changes —
  extra plumbing a simple deployment plausibly hasn't built. Absent that, freshness is bounded purely by
  the schedule interval, not by when the document actually landed: a corrected PDF that lands in Blob
  Storage five minutes *after* a scheduled run just finished sits there, fully re-indexable, invisible to
  the bot, for the entire length of the next interval — nearly 4 hours of the bot confidently citing the
  old text, with nothing in the architecture surfacing that gap even exists.

  `mergeOrUpload` itself is a specific, real operation, and its semantics are the crux of the next
  bullet: for each document key in a batch, it either **merges** fields into an existing index entry
  with that key, or **inserts** a new entry if the key doesn't exist yet. Which one happens is
  determined entirely by whether the key was already present — which is exactly why the key's design is
  load-bearing.

- **Chunk IDs are derived from `document_id` + chunk index**, not from a stable logical-document
  identity — something like `chunk_id = f"{document_id}_{chunk_index}"`. That's a reasonable design
  *if* `document_id` is stable across the life of "this policy document." It isn't, per course 05's
  confirmed finding: *every upload — including a revision of an existing document — gets a brand-new
  `document_id`* (chapter 05 of course 05; there's no `document_group_id`/`supersedes_document_id` in
  the real system, only a proposed one). Each upload is just a fresh, unrelated row from the uploader's
  point of view — nothing records that one document is a revision of another.

  Walking through it concretely:

  1. **Day 1** — "Remote-Work-Policy.pdf" is uploaded, gets `document_id = doc-123`. The indexer runs
     and chunks it into keys `doc-123_0`, `doc-123_1`, `doc-123_2`.
  2. **Day 30** — someone corrects a paragraph and re-uploads the same logical policy. The uploader
     assigns `document_id = doc-456`, because nothing in the data model knows this is a revision of
     `doc-123` — it's indistinguishable from uploading an unrelated new document.
  3. **Next indexer run** — `mergeOrUpload` sees keys `doc-456_0`, `doc-456_1`, `doc-456_2`. None of
     those keys exist yet, so every one is an **insert**, not a merge. Nothing in that operation touches
     `doc-123_0..2` — there's no relationship between the two key prefixes for `mergeOrUpload` to act on.

  End state: the index permanently holds **both** sets of chunks side by side, with no field marking
  either one as superseded. A cleanup filter like `document_id eq 'doc-123'` still works *if you already
  know* `doc-123` is the stale one — but the index itself has no way to tell you that; it just holds two
  equally well-formed, equally "valid" sets of chunks about the same policy, one of them silently wrong.

- **The practical result — and why it's worse than plain staleness:** this doesn't just clutter the
  index, it actively corrupts retrieval, and in a way that's hard to notice. Hybrid search (vector +
  keyword) scores every chunk against the query independently — it has no notion of "these two chunks
  are competing versions of the same fact." Both `doc-123_1` (old wording) and `doc-456_1` (new wording)
  are legitimate semantic matches for the same question, often with heavy literal overlap, so whichever
  one scores marginally higher in that specific ranking — which can hinge on phrasing coincidence rather
  than correctness — is what gets fed to the model as grounding. That makes the failure mode
  **nondeterministic per query**, not a clean "always stale" bug: the identical underlying staleness can
  produce a correct answer for one phrasing of a question and a confidently wrong, fluently-cited answer
  for a slightly different phrasing of the same question, with no exception thrown and nothing in the
  logs flagging that two competing chunks even existed.
- **A cache layer compounds the problem, not just the index.** A production chatbot under real load
  plausibly caches at two levels to control latency and Azure OpenAI cost: an **embedding cache**
  (avoid re-embedding the same chunk text across indexer runs) and a **full-answer cache** for
  frequently-asked questions (avoid a full retrieval-plus-generation round trip for "what are your
  branch hours" asked for the thousandth time). Neither of those has any hook into "a source document
  changed" — they're keyed on content/question text and expire on a timer, not on a freshness event.
  So even in the (unlikely) case where the index itself is somehow perfectly current, a full-answer
  cache entry populated before a correction shipped can keep serving the old, wrong answer verbatim
  until its TTL lapses.

The honest first sentence of the interview answer: *"Today, refreshing the index is scheduled, not
triggered by a document change — and there's no supersede signal anywhere in the pipeline, so a
revision doesn't replace the old chunks, it just adds new ones alongside them. On top of that, any
answer-level cache has no idea a source document changed at all, so it can keep serving a cached wrong
answer even after the index itself is fixed."*

## Part 2 — The manual workaround, achievable today

Exactly like course 05's manual "search, remove, re-upload" pattern, nothing here is fundamentally
impossible with what already exists — it's just entirely operator-driven, not automated or enforced:

1. **Identify the stale document's `document_id`.** This requires manual coordination with whoever
   owns the upload side — course 05's chapter 05 workaround already produces this exact ID as a
   byproduct of its own "search by title, note the old `document_id`" step, which is a nice, concrete
   point of contact between the two systems.
2. **Delete the stale chunks from the index by filter.** Azure AI Search supports deleting documents
   (in the search-index sense — "documents" here means indexed chunks, not the source PDFs) via a
   filter expression, e.g. `document_id eq '<stale-id>'`, through the index's REST/SDK delete-by-query
   path. This removes every chunk belonging to the old revision in one call, without needing to know
   individual chunk IDs.
3. **Force an out-of-cycle indexer run** (or wait for the next scheduled one) so the new revision's
   chunks are in place.
4. **Manually bust the answer cache**, if one exists — in the simplest real deployment, this could be
   as blunt as a debug/admin endpoint that clears the in-process dict, or restarting the App Service
   instances (which also, not coincidentally, clears anything else living in per-process memory — see
   Chapter 7's discussion of exactly what else lives there).

This is a genuinely useful thing to say plainly in an interview: *"the individual steps all exist and
work today — delete-by-filter is a real Azure AI Search capability, forcing a re-index is a real
operation, clearing a cache is a real (if blunt) operation — the gap is that nothing connects them
automatically. A person has to know a document was revised, know its old `document_id`, and remember
to run all three steps in order, and nothing stops them from skipping one and leaving a stale chunk or
a stale cached answer live in production."*

## Part 3 — Two concerns that are easy to conflate: conversation turn count vs. knowledge freshness

This is the distinction worth drawing precisely, the same way course 05's chapter 05 draws a clean line
between "approval lifecycle" and "content revision." Here, the two concepts that sound similar — both
involve something going "stale" — but are **completely orthogonal axes**:

| | **Conversation turn count** (Chapter 3) | **Knowledge freshness** (this chapter) |
|---|---|---|
| What it governs | How much of the *running dialogue* is included in the prompt | Whether the *retrieved context* reflects the current version of the source documents |
| Scope | Per-conversation, per-session | Global — shared by every conversation, every user |
| The failure mode | The bot "forgets" something the user said 20 turns ago because it fell out of the sliding window / token budget | The bot confidently answers from a policy that was corrected yesterday, because the index (or a cached answer) still has the old text |
| What fixes it | A sliding window or summarization strategy (Chapter 3) | Index invalidation on document change, cache invalidation tied to that same event (Part 4, below) |
| What does *not* fix it | — | **Starting a brand-new conversation does not fix a knowledge-freshness problem.** A user with zero prior turns, asking the very first question of a fresh session, gets exactly the same stale answer as a user 40 turns into a long conversation — because the staleness lives in the index/cache, not in anything conversation-scoped. |

That last row is the concrete, testable way to prove the two are unrelated, and it's worth stating
explicitly if asked to distinguish them: *"If someone suggested 'just clear the conversation history and
that'll fix it,' the answer is no — clearing history only affects what the model sees of the dialogue so
far, it has zero effect on what the retrieval layer returns for the next question, stale or not.
Conversely, a conversation could run for 100 turns discussing a document that never changed, and every
one of those turns would still be accurately grounded, because nothing about the retrieval side degraded
just because the conversation got long."* Getting this distinction right, unprompted, is exactly the
kind of signal that shows you're not pattern-matching "something says stale" to a single fix.

A closely related but distinct third axis, worth naming if the conversation goes deeper (Chapter 7
covers it in operational terms): **prompt/config version skew across a fleet of App Service
instances.** If the system prompt or a retrieval parameter is loaded from a config store at process
startup and cached in-process, a mid-day change (say, compliance requires an immediate wording
correction) doesn't propagate until every instance restarts or reloads — during a rolling deployment,
some instances serve the old prompt version and some serve the new one, simultaneously, in production.
That's neither a conversation-memory problem nor an index-freshness problem — it's a *third* kind of
staleness (configuration, not content, not dialogue), and conflating it with either of the first two is
just as easy a mistake to make under interview pressure.

## Part 4 — A genuine proposed design for real freshness

Explicitly a **proposal**, not something implemented, exactly in the spirit of course 05's chapter 05.
The fix has three coordinated pieces:

**1. An event, not a schedule, as the trigger.** If course 05's proposed `document_group_id` /
`supersedes_document_id` fields (course 05, chapter 05, Part 4) were ever actually added to the
uploader service's data model, the write of a non-null `supersedes_document_id` is a natural,
already-available trigger point: the moment that write happens, the uploader service knows, with
certainty, that document `B` supersedes document `A`. Publishing a small event at that moment —
plausibly an Azure Service Bus message or an Event Grid event —

```python
# Illustrative -- proposed, not implemented anywhere today
{
    "event": "document_superseded",
    "old_document_id": "A",
    "new_document_id": "B",
    "document_group_id": "<shared-group-id>",
    "business_line": "IWPB",
    "occurred_at": "2026-07-23T09:04:11Z",
}
```

— turns "a document was revised" from a fact only the uploader service knows into a fact the whole
downstream pipeline can react to, immediately, instead of waiting for the next scheduled indexer run.

**2. A small index-sync worker that consumes that event and does three things:**

```python
# Illustrative -- proposed worker logic, not implemented
def on_document_superseded(event: dict) -> None:
    old_id = event["old_document_id"]

    # (a) Retire the old chunks from the retrieval index immediately,
    #     rather than waiting for the next scheduled indexer run.
    search_index.delete_by_filter(f"document_id eq '{old_id}'")

    # (b) Invalidate any cached embeddings tied to the old document's chunks --
    #     keyed by content hash + embedding model version (see below), so this
    #     is a targeted purge, not a full cache flush.
    embedding_cache.evict_prefix(f"emb:{old_id}:")

    # (c) Invalidate any full-answer cache entries whose citations included the
    #     old document -- requires a small reverse index (doc_id -> cache keys
    #     that cited it), maintained whenever an answer is cached with its
    #     citation list attached.
    for cache_key in answer_cache_citation_index.get(old_id, []):
        answer_cache.evict(cache_key)
```

**3. Cache keys designed to fail safe by default, as a second line of defense** behind the event above
(events can be missed — a worker can be down, a message can be dropped — so the cache design itself
should minimize blast radius even with zero invalidation events firing):

- **Embedding cache**: key on `sha256(chunk_text) + embedding_model_version`, not on `document_id`
  alone. This means a chunk's embedding is naturally content-addressed — if the text changes at all,
  the key changes, and the old entry simply becomes unreferenced (still technically present, but never
  looked up again) rather than actively serving wrong data. It also means an embedding-model upgrade
  invalidates every cached embedding automatically, for free, without a manual purge step. TTL here can
  be long (weeks) since a stable key is safe to keep — the risk isn't staleness, it's just storage.
- **Full-answer cache**: a much shorter TTL by design — plausibly 10–15 minutes rather than hours —
  precisely *because* this is the layer with no natural content-addressing (a question's phrasing
  doesn't change when the answer to it does), so time-based expiry is the only safety net when the
  event-driven purge in step 2 doesn't fire for some reason.
- **Surface the freshness signal to the user, not just fix it silently.** Attaching an
  `indexed_as_of` timestamp to each retrieved chunk (or each citation shown in the UI) lets a user or
  reviewer visually judge whether an answer might be running on stale grounding, the same instinct
  behind course 05's audit-trail thinking — a system that's honest about what it doesn't know it's
  missing is more trustworthy than one that silently might be wrong.

## Part 5 — The forward-looking tie: this reaches three systems, not one

This is worth volunteering, the same way course 05's chapter 05 volunteers the retrieval-index tie —
it demonstrates the fix isn't "add a column," it's "understand the whole pipeline this has to travel
through":

- **The Document Uploader Service (course 05) would need to actually emit the event.** Adding
  `supersedes_document_id` to its data model (course 05's own proposal) is necessary but not
  sufficient — the write has to be paired with publishing the event above, which is new integration
  work in a system that today has no outbound eventing at all for document lifecycle changes.
- **Azure AI Search needs the delete-by-filter call wired into an automated consumer**, not a manual
  operator step — the capability already exists in the platform, the gap is purely that nothing
  invokes it automatically today.
- **Any answer-level cache needs the same invalidation hook**, or a short-enough TTL to bound the
  damage in the meantime — and if the chatbot is horizontally scaled across multiple App Service
  instances (Chapter 3, Chapter 7), that cache almost certainly needs to move out of per-process memory
  into a shared store (Redis) for the invalidation event to even have a single place to take effect,
  rather than needing to fan out to N independent in-process caches that can each miss the message.

The precise way to state the overall shape of the fix, if asked to summarize it in one breath:
*"the trigger has to move from a schedule to an event, the event has to originate where the truth
actually lives — the uploader service, at the moment it records a supersede — and it has to reach every
place that could still be holding onto the old content: the search index, the embedding cache, and the
answer cache, in that order, because fixing only the first without the other two just moves where the
stale answer comes from, it doesn't remove it."*

## Tying It Back

If asked "how do you handle it when the source documents behind this chatbot's answers get updated or
corrected" — this chapter is the fully worked answer: today's gap is real and specific (schedule-driven
refresh, no supersede signal, cache with no freshness awareness), the manual fix is genuinely usable
right now with existing Azure AI Search capabilities, the turn-count/freshness distinction is a clean,
testable line worth drawing unprompted, the proposed event-driven design is concrete enough to sketch on
a whiteboard, and the honest scope of that fix reaches three separate systems — the uploader service,
the search index, and the cache layer — not just this chatbot's own code.
