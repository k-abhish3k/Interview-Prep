# 08 — Vector Index Staleness and Document Revision Handling

## Why this chapter exists

**A note on grounding before anything else**: unlike course 5 (the Capco Document Uploader Service),
which was rebuilt directly against real, proprietary source code, this course has no real source
repository behind it — it's a technically detailed, plausible reconstruction built from the resume
bullet and the production facts in the root `README.md` (Eli Lilly/AstraZeneca, AWS ECS Fargate,
Pinecone namespaced per client). Nothing in this chapter is a verified fact about Indegene's actual
implementation. It's written with the same confidence and specificity as course 5's equivalent
chapter because that's the right way to *reason* about the problem and the right way to *answer* the
interview question it's built around — but the specific mechanism names, field names, and code below
are illustrative engineering judgment, not a description of code that exists.

That interview question is the direct analog of the one that triggered course 5's chapter 05: **"When
a project's data changes — a cost catalog entry gets repriced, a project's status moves, a document
gets revised — how do you make sure the chatbot doesn't keep retrieving and answering from the old,
stale vector in Pinecone?"** It's the same underlying failure mode as course 5's "revised documents
aren't versioned" gap, just one layer down the stack: course 5 asks whether the *document store*
knows which upload is current; this chapter asks whether the *retrieval index* built on top of that
kind of data knows which vector is current. A RAG platform that gets document versioning right at the
source-of-truth layer but never propagates that signal into Pinecone has solved half the problem and
left the half a client actually experiences — a wrong answer, delivered fluently and confidently by an
LLM that has no idea the chunk it was handed is stale.

## Part 1 — What a realistic implementation does today

A Virtual Liaison-style platform's Pinecone index isn't populated once and left alone — project status
changes daily, cost-catalog prices get revised, documents get replaced. A realistic design keeps the
index in sync with two complementary mechanisms:

1. **An upsert-on-change trigger from the source system.** When a project's status changes in the
   project-management system, or a cost-catalog SKU is repriced, that system emits a change event
   (an internal event bus message, a webhook, or a row in a changes-since-last-sync table) that a sync
   worker consumes. The worker re-chunks and re-embeds the changed content and calls `index.upsert(...)`
   using a **stable, deterministic chunk ID** — typically derived from `(client_id, project_id,
   doc_type, chunk_index)` rather than a random UUID — so that a status change to the same project
   overwrites the existing vector in place, exactly as Chapter 2 describes: *"the chunk representing
   'current status' gets re-embedded and upserted under the same ID rather than growing an
   ever-longer history."*
2. **A periodic full re-index job as a backstop.** Nightly (or on some other fixed cadence), a batch
   job walks every currently-live project and catalog entry, re-chunks and re-embeds all of it, and
   re-upserts under the same deterministic IDs. This exists specifically to self-heal drift the
   event-driven path missed — a dropped event, a sync worker outage, a bug in the change-detection
   logic — at the cost of the batch job's own latency: anything wrong is only guaranteed to be caught
   at the next nightly run, not the moment it happens.

**The honest gap, worth naming plainly rather than glossing over: there is no symmetric "content
removed" path.** The upsert-on-change trigger fires when a project's *fields change* — it has no
counterpart that fires when a project is **deleted, cancelled, or archived** in the source system, or
when a cost-catalog SKU is **discontinued**. Concretely: if Project Atlas is cancelled and removed from
the project-management system, nothing calls `index.delete(...)` for its vectors. They simply remain
in Pinecone, indefinitely, fully retrievable — because nothing in the sync design distinguishes
"this project's data changed" from "this project no longer exists," and the nightly re-index job is
built to re-upsert what's currently *live*, not to diff against what's already in the index and delete
what's fallen out. A client could ask "what's the status of Project Atlas" months after it was
cancelled and get a confident, grounded-sounding answer synthesized from a vector that represents a
project that, as far as the system of record is concerned, doesn't exist anymore.

## Part 2 — The manual workaround, achievable today without a redesign

Nothing about this gap is unfixable in the moment — it's just entirely reactive and human-driven,
the same shape as course 5's manual supersede workaround:

1. **Someone notices.** Typically a client liaison spot-checking an answer, or a QA reviewer running
   the kind of golden-query regression check described in Chapter 9, notices the assistant citing a
   project or catalog item that shouldn't exist anymore.
2. **An engineer confirms in the source system** that the project really is gone (not just delayed or
   renamed — a real risk in a fast-moving pharma engagement where "cancelled" and "paused pending legal
   review" can look similar from the outside).
3. **An engineer manually issues a scoped delete**: `index.delete(filter={"project_id": {"$eq":
   "atlas"}}, namespace="eli-lilly")` (or the equivalent explicit-ID form), run by hand against the
   correct client's namespace.

This is a genuinely useful, precise thing to say in an interview: *"the tooling to fix this already
exists — Pinecone's delete-by-filter API does exactly what's needed — the gap is that nothing triggers
it automatically."* It's also worth naming the workaround's own real risk, honestly: a Pinecone delete
is not soft or reversible the way a database soft-delete flag is, and a delete-by-filter call scoped to
the wrong namespace, or a filter that's broader than intended, permanently removes vectors with no
built-in undo. That's a "measure twice, cut once" caveat worth stating candidly rather than presenting
the manual workaround as risk-free just because it's simple.

## Part 3 — Two concerns that are easy to conflate, and shouldn't be

This is the distinction worth being precise about under interview pressure, because it's exactly the
kind of nuance that separates "I used a vector database" from "I understand what a vector database
actually guarantees":

- **"The vector's similarity score is still numerically valid."** Nothing is wrong with the ANN math.
  If a cancelled Project Atlas's status-update vector is still sitting in the index, and a client asks
  about Project Atlas, that vector may genuinely be the closest vector in the index to the query
  embedding — the cosine similarity score Pinecone returns is completely correct, computed honestly
  against real data. Retrieval did its job.
- **"The underlying content it represents is no longer current or true."** That's a completely separate
  axis, and Pinecone has no native concept of it at all. A vector store's similarity metric measures
  *semantic closeness between a query and a piece of text*, full stop — it has no built-in notion of
  "freshness," "supersession," or "still-exists-in-the-source-system" unless that information is
  explicitly encoded as metadata and explicitly filtered on at query time. A stale vector doesn't
  self-flag as stale; it can score *higher* than a fresher, more relevant one purely because of
  incidental wording overlap, and nothing about the retrieval mechanics would ever surface that as an
  anomaly.

The precise way to state this in an interview: *"A high similarity score tells you the retrieval math
worked — it does not tell you the content is still true. Those are two different questions, and a RAG
system that only monitors retrieval quality (are we getting relevant-looking chunks) can look perfectly
healthy by that measure while quietly grounding answers in dead data. Staleness has to be solved as a
data-freshness problem, with its own explicit signal — a version or currency flag — not assumed away
because retrieval 'looks like it's working.'"** This is the same category of distinction course 5's
chapter 05 draws between IWPB's approval state machine and document versioning — two axes that look
related, aren't, and get dangerously conflated if you're not deliberate about naming them separately.

## Part 4 — A proposed design for real version-awareness

This is explicitly a **proposal** — a defensible design for closing the gap named in Part 1, not a
claim about what's implemented. It needs three pieces, and the two-layer distinction from course 5's
chapter 05 (this service's data model vs. the downstream system of record) applies here too, just
relocated: the "data model" layer is the vector's own metadata, and the "downstream system" is the
query path every retrieval-based feature shares.

**1. Version metadata on the vector itself.** Add three fields to every vector's metadata, alongside
the existing `client_id` / `project_id` / `doc_type` (Chapter 2):

```python
# Proposed metadata additions — illustrative, not implemented today
metadata = {
    "client_id": "eli-lilly",
    "project_id": "atlas",
    "doc_type": "status_update",
    "document_version": 3,            # increments every time this logical chunk is re-embedded
    "supersedes_id": "atlas-status-02-v2",  # the prior version's vector ID, null if this is v1
    "is_current": True,               # flipped to False the moment a newer version is upserted
}
```

`document_version` and `supersedes_id` are the same conceptual fields course 5's chapter 05 proposes
for the document store (`document_group_id` / `supersedes_document_id`), applied one layer down: there,
the goal is "which upload is the current revision of this logical document"; here, the goal is "which
vector is the current embedding of this logical chunk." `is_current` is the field that makes the fix
enforceable at query time, not just at write time.

**2. A re-index-and-retire write pattern, ordered to fail safe.** When a project's status changes (or
a document is revised):

```python
def upsert_new_version(new_content: str, prior_vector_id: str, chunk_id: str,
                        current_version: int, metadata_base: dict):
    new_vector_id = f"{chunk_id}-v{current_version + 1}"
    # Step 1: write the new version FIRST.
    index.upsert(vectors=[{
        "id": new_vector_id,
        "values": embed(new_content),
        "metadata": {
            **metadata_base,
            "document_version": current_version + 1,
            "supersedes_id": prior_vector_id,
            "is_current": True,
        },
    }], namespace=metadata_base["client_id"])

    # Step 2: retire the old version ONLY after the new one is confirmed written.
    index.update(id=prior_vector_id, set_metadata={"is_current": False},
                  namespace=metadata_base["client_id"])
    # or, if the design prefers a hard delete over an audit trail:
    # index.delete(ids=[prior_vector_id], namespace=metadata_base["client_id"])
```

The ordering is deliberate, not incidental: **upsert-new-then-retire-old**, never the reverse. Pinecone
has no cross-document transaction — there's no way to make both writes atomic — so if the process
crashes between the two steps, upsert-first means the failure mode is "both versions are briefly
marked current" (retrievable, survivable, resolved by the next reconciliation pass), not "a window
where neither version is current and the client's question gets no grounding at all." A soft-delete
(`is_current: False`) is generally preferable to a hard `index.delete` for exactly the audit-trail
reason course 5's chapter 05 raises about losing a document's ID on remove-and-reupload: keeping the
superseded vector queryable-by-explicit-ID (just excluded from normal retrieval) preserves a "what did
we used to say" history that a hard delete throws away.

**3. A "get latest version only" filter, applied at every retrieval call, as defense in depth.** Every
query against project or catalog data — not just the obvious ones, every single retrieval code path,
including the recommenders in Chapter 6, which reuse the same index — adds `is_current` to its existing
filter:

```python
def retrieve_for_client(query_text, client_id, project_id=None, doc_type=None, top_k=5):
    filt = {"is_current": {"$eq": True}}
    if project_id:
        filt["project_id"] = {"$eq": project_id}
    if doc_type:
        filt["doc_type"] = {"$eq": doc_type}
    return index.query(vector=embed(query_text), top_k=top_k,
                        namespace=client_id, filter=filt, include_metadata=True)
```

This is the same "don't rely on one clever trick" philosophy Chapter 2 uses for namespace isolation,
applied to staleness: even if step 2 of the write pattern above fails silently, or hasn't run yet
because of replication lag, or a future engineer adds a new retrieval code path and forgets the
convention, the `is_current` filter is the backstop that keeps a superseded vector from ever reaching
an LLM's context window. It also directly closes the Part 1 gap for **deletions**: retiring a cancelled
project's vectors becomes exactly the same `is_current: False` write the revision path already does —
"this project was deleted" and "this project was revised" both reduce to "flip `is_current` on the
vectors that are no longer valid," which is a much smaller, more uniform surface to get right than
building a separate deletion pipeline from scratch.

## Part 5 — The forward-looking tie to summarization: why this can't be solved downstream

This is the point worth volunteering, because it demonstrates the same systems-level thinking course 5's chapter 05 closes with about a downstream search index: **Chapter 4's "concise project tracking"
summarization feature makes staleness worse, not just unhelped, if this isn't fixed at the retrieval
layer.** A raw retrieved chunk that's stale is at least inspectable — it typically carries a visible
date or status label a careful reader (or a citation-aware UI) could notice looks old. A map-reduce or
refine summarization chain (Chapter 4) takes that same stale chunk and **launders it into fluent,
confident prose** — "Project Atlas is on track for a Q3 Japan launch" reads exactly as authoritative
whether it was synthesized from a current status update or a cancelled project's last-known state
before someone forgot to tell the index. The summarization step doesn't introduce the staleness, but it
actively removes the reader's best chance of noticing it, because synthesis strips away the raw,
dated, individually-inspectable evidence in favor of a single smooth narrative.

The concrete implication: the `is_current` filter from Part 4 has to sit **upstream of** Chapter 4's
summarization chain, not be treated as a parallel, independently-schedulable improvement. If the
version-aware retrieval fix here isn't in place, making the summarization chain itself "smarter" about
staleness (e.g., asking the LLM to hedge when it's unsure) is solving the wrong layer — you'd just be
summarizing garbage more fluently and more persuasively. This is the same shape of point course 5
makes about a downstream search index needing to retire a superseded document's entry: the fix has to
reach as far downstream as the content itself does, and in this platform, summarization is exactly that
downstream consumer.

## A secondary angle worth naming: conversation-session staleness

The vector-index case above is the primary, cleanest analog to course 5's chapter 05, but it's not the
only place a "numerically valid but no longer current" mismatch can show up in this platform. Chapter
7's LangGraph orchestration layer checkpoints conversation state (`LiaisonState`) to an external store
so any ECS task can resume a session. If a client's session is checkpointed mid-conversation — say,
partway through a multi-turn "I need a new localization, what would that cost" exchange — and the
underlying project or catalog data changes materially before the client's next turn arrives (a long
pause, a dropped connection, a voice call that resumes minutes later), the **checkpointed state itself**
can reference facts that were true when it was saved and aren't anymore, independent of whether the
vector index has been kept current. The practical mitigation is narrower than the retrieval-layer fix
above — checkpointed state that carries forward a specific fact (a quoted price, a stated status) worth
a freshness check should be re-validated against a fresh retrieval call before being restated to the
client, rather than replayed verbatim from checkpoint, especially across a gap long enough that the
underlying data plausibly changed. This is worth raising as a secondary point if an interviewer pushes
past the Pinecone answer with "is that the only place staleness can hide" — it demonstrates the same
concept applies at the orchestration layer, not just the retrieval layer, without overclaiming it's
been solved there.

## The full interview-ready answer, assembled

> "There are two mechanisms that keep Pinecone roughly in sync with the source systems: an
> upsert-on-change trigger that re-embeds and re-upserts under a stable ID whenever a project's status
> or a catalog price changes, and a periodic full re-index as a backstop for anything that trigger
> missed. The honest gap is that there's no symmetric path for *removed* content — if a project gets
> cancelled or a SKU discontinued, nothing calls delete, so those vectors just sit in the index
> indefinitely and stay fully retrievable. Today, the fix is entirely manual and reactive: someone
> notices a stale answer, confirms it in the source system, and an engineer runs a scoped
> delete-by-filter call by hand — the tooling exists, nothing triggers it automatically.
>
> It's worth being precise that a stale vector isn't a retrieval bug — its similarity score is
> completely valid; the model is correctly finding the text closest to the query. The problem is a
> separate axis entirely: whether that text is still true, which Pinecone has no native concept of at
> all unless you encode it. If I were closing this gap for real, I'd add version metadata to every
> vector — a `document_version`, a `supersedes_id` pointing at the prior version, and an `is_current`
> flag — write new versions before retiring old ones so a crash mid-update never leaves a client with
> zero grounding, and add `is_current: true` to every retrieval filter as defense in depth, so even a
> missed retire step can't surface stale content. That same flip — `is_current: false` — is also exactly
> how I'd close the deletion gap, since 'this project was cancelled' and 'this project was revised' both
> reduce to the same operation.
>
> And this has to be fixed upstream of summarization, not alongside it — a raw stale chunk at least
> looks inspectable, but a summarization chain turns it into fluent, confident prose that hides exactly
> the cues that would let someone notice it's out of date. Fix retrieval first; summarization inherits
> the fix for free."

## Tying It Back

This chapter is the vector-index-layer restatement of course 5's document-versioning question, and the
parallel is deliberate: both answers have the same shape — name plainly what's not automated today,
describe the manual workaround that already exists with existing tools, draw a precise line between two
concepts that are easy to conflate, propose a genuine versioning design, and tie it forward to whatever
downstream system would otherwise keep serving the stale content. The strongest part of this answer, as
with course 5, isn't the proposed `is_current`/`supersedes_id` design in Part 4 — it's Parts 1 and 3:
naming the real gap (no delete-on-removal path) honestly, and drawing the clean distinction between "the
similarity math is correct" and "the content is current," so an interviewer can see those two ideas
aren't being confused under pressure. `notebooks/08_vector_version_aware_upsert_and_retire.ipynb`
implements this exact upsert-new-then-retire-old pattern against the same `FakePineconeIndex` stand-in
from Chapter 2's notebook, and demonstrates a query that returns stale content before the fix and correct
content after.
