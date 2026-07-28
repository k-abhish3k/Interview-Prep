# 05 — Document Lifecycle, Versioning, and Revisions

## Why this chapter exists

This chapter exists because of a real interview question the candidate was actually asked about this
project: **"for revised versions of the same document, how are you handling those?"** This is the
chapter to have fully internalized before that question comes up again — it's structured as the
answer itself, not just background reading.

The honest answer has five parts: what happens today (nothing — every upload is a new document), a
manual workaround achievable right now with existing endpoints, why IWPB's approval workflow is a
different concern entirely and shouldn't be confused with versioning, a genuine proposed design for
real revision tracking, and the forward-looking tie to retrieval that shows systems-level thinking.

## Part 1 — What happens today: there is no versioning

This is confirmed directly from reading `upload_files()` in `app.py` — there is **zero** title-based
deduplication, "supersedes" logic, or version linkage anywhere in the upload path, for any department.
Every single call to `POST /upload-files/{use_case}` — IWPB included, once approved — results in:

1. A `POST` to `INGEST_API`'s `ingest/batch-initialize/{use_case}/HEXA`, which returns a **brand-new**
   `document_ids[]` for the batch, with no lookback at whether a document with the same title already
   exists.
2. A `POST` to `INGEST_API`'s `ingest/{use_case}/HEXA` per file, returning a **brand-new** `documentID`.

There is no query anywhere in this codebase that asks "does a document with this title/business line
already exist" before creating a new one. Uploading the exact same file, with the exact same title,
to the exact same business line, twice in a row, produces **two entirely independent documents** in
HEXA with two different `documentID`s — the second upload has no awareness the first one exists.

This is the direct, honest first sentence of the interview answer: *"Today, there's no versioning —
every upload, from any department, creates a brand-new document with a brand-new ID in the ingestion
system. If you upload a revised version of a policy PDF with the same title, it doesn't overwrite,
link to, or supersede the original — it's a completely separate document, and both are discoverable
through search unless someone manually removes the old one."*

## Part 2 — The manual "supersede" workaround, achievable today

Nothing in the current API *prevents* the correct end result — it's just entirely client-driven, not
server-enforced. The existing endpoints compose into a working (if manual) three-step pattern:

1. **Find the stale version.** `GET /search-ingested-documents/{use_case}?document_title=<title>` — the
   endpoint already accepts a `document_title` filter (`app.py`'s `search_ingested_documents`, proxying
   to `INGEST_API`'s `GET ingest/HEXA`), so a client can look up any existing document(s) with a matching
   title in the target business line before uploading a new one.
2. **Remove the stale version.** `POST /remove-ingested-document/{use_case}` with the stale document's
   `parent_document_id` — this already exists and already works (`DELETE ingest/{use_case}/HEXA` on
   `INGEST_API`, with `strict=true`).
3. **Upload the revision.** `POST /upload-files/{use_case}` as normal — the revised file gets a fresh
   `documentID`, and because the old one was removed in step 2, there's no longer a stale duplicate
   sitting in HEXA (or in whatever downstream search index HEXA feeds).

This is a genuinely useful thing to be able to describe confidently: *"the pieces to do this today
already exist and already work — search by title, remove the old one, upload the new one — the gap
isn't that it's impossible, it's that nothing enforces or automates it. A client (or a user) has to
know to do all three steps, in order, and nothing stops them from skipping step 2 and leaving a stale
duplicate behind."* That's a precise, defensible statement of the actual gap, not a vague "we don't
support that."

**Caveat worth stating alongside this:** this manual workaround loses the original document's
`documentID` — anything that referenced the old ID (a link, a citation in a chat answer, an audit log
entry) breaks, because "remove and re-upload" is not the same operation as "create a new version of
the same logical document." That's exactly the gap Part 4 below is designed to close.

## Part 3 — Why IWPB's status machine is a different concern, not versioning

It's tempting to point at IWPB's `PENDING_APPROVAL → APPROVED | DECLINED | AUTO_REMOVED; APPROVED →
PURGED` state machine and describe it as "how we handle document lifecycle" in a way that sounds like
it answers the versioning question. It doesn't, and conflating the two is a mistake worth actively
avoiding:

- **IWPB's status machine is an approval/retention lifecycle for a single document instance.** It
  tracks whether *this one upload* has been reviewed, approved, declined, or has expired — not whether
  it's the latest revision of some logical document that's had multiple versions over time.
- **There is no concept of "this APPROVED document was superseded by that later APPROVED document"**
  anywhere in `IWPBDocumentWorkflow`. Uploading a revised version of an already-approved IWPB document
  creates an entirely new `workflow_id`, goes through its own independent `PENDING_APPROVAL` cycle, and
  has zero link back to the row it's meant to revise.
- **`PURGED` is expiry-driven, not revision-driven.** A document reaches `PURGED` because its
  `expiry_date` arrived and the background sweep removed it from HEXA — not because a newer version of
  it was uploaded. A document could be revised the day after upload and both the original and the
  revision would independently run out their own expiry clocks, with no relationship between them.

The precise way to draw this boundary in an interview: *"IWPB's status machine answers 'has this
specific upload been approved, and is it still within its retention window' — it's an approval and
expiry lifecycle for one document instance. It doesn't answer 'is this the current version of a
document that's had several revisions' — that's a different axis entirely, and today, nothing in the
system tracks that second axis at all, for IWPB or any other department."*

## Part 4 — A genuine proposed design for real revision tracking

This is explicitly a **proposal**, not something implemented — labeled as such throughout. A real
revision-tracking feature would need changes at two layers:

**This service's data model.** Add two nullable fields to the concept of "a document" (today, that
concept only has real local state for IWPB, in `IWPBDocumentWorkflow` — but the same fields generalize
to any future per-document tracking table):

```python
# Proposed additions — not implemented today
document_group_id = sqla.Column(sqla.String(36), nullable=True)
# Stable identifier shared by every revision of "the same logical document."
# Generated fresh on first upload; carried forward by the client (or resolved
# server-side via a title+business_line lookup) on every subsequent revision.

supersedes_document_id = sqla.Column(sqla.String, nullable=True)
# The HEXA documentID (or workflow_id, for IWPB pre-approval) of the specific
# prior revision this upload replaces, if any. Nullable because the first
# upload of a document has no predecessor.
```

"Get latest version" then becomes a defined query rather than an ambiguous title search: *the row with
the given `document_group_id` that has no other row pointing at it via `supersedes_document_id`* (or,
more simply, the most recently created row in that group whose status is still active/approved). A
version history view is just "every row sharing this `document_group_id`, ordered by `created_at`" —
both queries are straightforward once the two fields exist, and neither is possible today with zero
linkage between uploads.

**`INGEST_API`/HEXA itself would need to gain a version concept.** This is the part worth being
explicit about as a genuine system boundary, not a detail this service can paper over: `INGEST_API`
today has no `supersedes` or `document_group_id` parameter on `ingest/batch-initialize` or `ingest`, and
`GET ingest/HEXA` has no "give me only the latest revision per group" query mode. This service could
track `document_group_id`/`supersedes_document_id` locally (in its own SQL table) without any HEXA
change, but that only gets you "this service knows which uploads are revisions of each other" — it
does **not** by itself stop HEXA (or anything reading from HEXA directly, bypassing this service) from
still surfacing the stale revision. A complete implementation needs the downstream system of record to
understand versioning too, which is genuinely outside this service's control and would need to be a
cross-team conversation with whoever owns `INGEST_API`/HEXA, not something this service can
unilaterally fix.

## Part 5 — The forward-looking tie to retrieval: why this matters beyond bookkeeping

This is the point worth volunteering to demonstrate systems-level thinking, not just data-modeling
skill: if HEXA (or a downstream RAG/search layer built on top of it) maintains per-department search
indexes — the candidate's broader description of "4 different Azure AI Search indexes per department"
is plausible here even though it isn't visible in this repository (see chapter 03's honest scope
boundary on that) — then **a real revision feature has to do more than add a database column.** When
document B supersedes document A, the old index entry for A needs to be retired (or explicitly marked
stale/excluded from retrieval), or a RAG system built on top of that index keeps serving the
superseded content indefinitely, potentially alongside the new revision, with no signal to the retrieval
layer or the end user about which one is authoritative. That's exactly the kind of follow-up point that
distinguishes "I'd add a column" from "I understand this needs to propagate through the whole pipeline,
not just this service's database" — the versioning fix has to reach as far downstream as the content
itself does.

## The full interview-ready answer, assembled

> "Today, there's no versioning in this system — every upload, in every department, creates a brand-new
> document with a brand-new ID; there's no title-based dedup or supersede logic anywhere in the upload
> path. What you *can* do today, with existing endpoints, is a manual three-step workaround: search by
> title and business line to find the stale version, remove it, then upload the revision — the pieces
> all exist and work, they're just not automated or enforced, and you lose the original document ID in
> the process. It's also worth being precise that this is a different concern from IWPB's
> approval/retention state machine — that tracks whether one specific upload has been approved and is
> within its retention window, not whether it's the current revision of a document that's had several
> versions. If I were building real revision tracking, I'd add a `document_group_id` and a
> `supersedes_document_id` to the document model, so 'get the latest version' and 'show me the revision
> history' become well-defined queries instead of ambiguous title searches — but that only solves half
> the problem, because the ingestion API this service calls doesn't have a version concept either today,
> so a complete fix needs that system to understand versioning too. And if there's a search index behind
> retrieval, which there likely is downstream of this, a real revision feature also needs to retire the
> old version's index entry when a new one supersedes it — otherwise you've solved the bookkeeping
> problem but a RAG system on top could still serve stale content."

## Tying It Back

This chapter is the direct, source-grounded answer to a question the candidate has already been asked
in a real interview. The strongest thing about this answer isn't the proposed design in Part 4 — it's
Parts 1 and 3: stating plainly, with a specific function name and a specific absence of logic, that
there is no versioning today, and drawing a clean, correct line between "approval lifecycle" and
"content revision" so the interviewer can see the candidate isn't confusing the two concepts under
pressure. Confidence about what's *not* built is just as strong a signal as knowing what is.
