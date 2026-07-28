# 09 — Development Challenge: The "Everything Eventually Needs a Revision" Pattern

## Why this chapter exists

This chapter exists because of a real, source-grounded finding, not a hypothetical. While rebuilding
course 5 (Document Uploader Service) directly from the candidate's actual proprietary source code, a
specific gap turned up: the service has **no built-in document-versioning or revision feature** —
every upload creates a brand-new document with a brand-new ID, and the only way to get a "revised
version replaces the old one" outcome today is a manual, three-step, entirely client-driven
workaround (search by title, remove the stale document, upload the revision). That finding is written
up in detail, with the exact function and endpoint names, in course 5's
[`05-document-lifecycle-versioning-and-revisions.md`](../05-Capco-Document-Uploader-Service/05-document-lifecycle-versioning-and-revisions.md).

What makes that finding worth a whole chapter here, rather than staying a course 5 footnote, is that
the same shape of gap shows up — independently, for domain-specific reasons — in every other project
in this curriculum. A document, a monitoring baseline, a deployed service version, a vector embedding,
an approved claim, a detection result, a clinical protocol: each one is an entity that gets superseded
by a newer version over time, and each project's original scope was built around *producing* that
entity, not around *tracking which version of it is current*. This chapter names that pattern
explicitly, so it's available as an answer for any of the nine projects, not just the one where it was
first noticed.

## Why this pattern recurs so often — it's scope, not carelessness

It would be easy to read "none of these nine systems has real versioning" as nine separate instances
of the same mistake. That's the wrong read, and saying it that way in an interview is actually a
weaker answer than the accurate one: **versioning is almost never in the original ticket.** The
original ask is always some variant of "ingest this document," "flag this claim," "detect this chart,"
"monitor this model," "generate this protocol section" — a single-pass transformation of one input
into one output. Nobody asks for "and also, handle what happens when a second, revised version of that
same input shows up next quarter" up front, because that requirement only becomes visible once the
system has been running long enough for a first revision to actually occur. It's not a bug in the
code that shipped; it's scope that was never asked for. Framing it that way — "this wasn't overlooked,
it was out of scope for what was actually asked" — is more credible under interview pressure than
either overclaiming ("we handle that") or over-apologizing ("we really should have built that"). Both
of those responses read as less senior than calmly stating what exists, what doesn't, and why.

## The pattern, named per course

| Course | Entity that gets superseded | Pointer |
|---|---|---|
| 3 — Document Uploader Service | The uploaded document itself | [`05-document-lifecycle-versioning-and-revisions.md`](../05-Capco-Document-Uploader-Service/05-document-lifecycle-versioning-and-revisions.md) — the source-grounded original finding |
| 1 — AI Chatbot Assistant | The RAG knowledge base / retrieval cache backing the chatbot's answers | This course's chapter on knowledge-base staleness (see course 1's chapter index) |
| 2 — Model Risk Monitoring | The monitoring baseline a model's live metrics are compared against | [`06-baseline-staleness-and-drift-lifecycle.md`](../04-Capco-Model-Risk-Monitoring/06-baseline-staleness-and-drift-lifecycle.md) |
| 4 — CI/CD Pipelines | The pinned version of a shared service/dependency across environments | [`06-multi-service-version-pinning-and-environment-drift.md`](../06-Capco-CICD-Pipelines/06-multi-service-version-pinning-and-environment-drift.md) |
| 5 — Superscript/Subscript Detection | The processed document/page run through the OCR pipeline | [`06-document-reprocessing-and-citation-deduplication.md`](../07-Indegene-Superscript-Subscript-Detection/06-document-reprocessing-and-citation-deduplication.md) |
| 6 — GenAI Virtual Liaison Platform | The Pinecone vector for a source document chunk | [`08-vector-index-staleness-and-document-revision-handling.md`](../08-Indegene-GenAI-Virtual-Liaison-Platform/08-vector-index-staleness-and-document-revision-handling.md) |
| 7 — Claim Extraction & Tagging | The approved-claims library entry a new claim is checked against | [`07-approved-library-versioning-and-stale-comparison.md`](../09-Indegene-Claim-Extraction-Tagging/07-approved-library-versioning-and-stale-comparison.md) |
| 8 — Chart/Graph Detection | The chart/graph detection result for a given source image | [`06-detection-deduplication-and-model-version-drift.md`](../10-Indegene-Chart-Graph-Detection/06-detection-deduplication-and-model-version-drift.md) |
| 9 — GenAI Regulatory Platform | The clinical protocol / SOC document and its amendments | [`06-protocol-amendment-versioning-and-document-revision-handling.md`](../11-Indegene-GenAI-Regulatory-Platform/06-protocol-amendment-versioning-and-document-revision-handling.md) |

Read each of those chapters for the domain-specific mechanics. This chapter is deliberately not
re-explaining any of them — it's the connective layer that lets you recognize, on the spot, that a
question about any one of them is the same underlying question.

## The general shape of a good on-the-spot answer

If an interviewer asks "how do you handle revisions of X" about a project you haven't specifically
rehearsed this angle for, the course 5 answer (Part 5 of chapter 05 there) is the template. It has a
fixed shape, regardless of which project X belongs to:

1. **Name what happens today, plainly.** Usually: nothing automated — a new upload, a new detection
   run, a new extracted claim, a new generated section, is treated as an independent unit with no
   awareness that an earlier version of the same logical thing exists.
2. **Name the manual workaround, if one exists.** Most of these systems already have the raw pieces
   (a search-by-identifier endpoint, a delete/remove endpoint, a re-run trigger) to reproduce a
   "supersede" outcome by hand — the gap is usually that nothing *automates or enforces* the
   sequence, not that it's structurally impossible.
3. **Explicitly separate the two concerns that get conflated.** Almost every one of these systems has
   *some* status/lifecycle machinery already (an approval workflow, a monitoring alert state, a
   detection confidence flag) that sounds like it answers the versioning question but doesn't: it
   tracks whether *this one instance* is technically valid, approved, or within a retention window —
   not whether it's still *current* or *true* relative to a newer instance. Course 5's IWPB approval
   state machine is the clean, documented example: "approved and not yet expired" and "the latest
   revision of this document" are two different axes, and a system can correctly answer the first
   while having no concept of the second at all.
4. **Propose a concrete mechanism.** The reusable shape is the same everywhere: a stable group/logical
   identity for "all versions of this same thing," a `supersedes`/`superseded-by` pointer from each
   version to the one before or after it, and a well-defined "get current version" query (the item in
   the group nothing else points at, or simply the newest active one) — not an ambiguous re-search by
   name or content every time. See `notebooks/04_generic_supersede_pattern_demo.ipynb` in this folder
   for a small, fully generic, runnable implementation of exactly that shape.
5. **Name the downstream system that also needs to know.** A version pointer added to one service's
   own database solves half the problem. Whatever consumes that entity downstream — a search index, a
   RAG retrieval layer, a dashboard, a compliance report, a training pipeline — keeps serving,
   comparing against, or training on the stale version until it's told about the new one too. This is
   the detail that most distinguishes "I'd add a column" from "I understand this has to propagate
   through the whole pipeline."

## Why this transfers better than one memorized answer

The value of internalizing this as a *pattern*, rather than memorizing course 5's specific answer, is
that it's genuinely transferable under pressure. An interviewer who asks "how do you handle a revised
version of X" for a project you haven't specifically prepped this angle for is testing exactly
whether you can reason about an unfamiliar system's data lifecycle live, not whether you've rehearsed
nine separate scripts. Walking through the five-step shape above — for a vector, a baseline, a claim,
a detection, a protocol, or anything else an interviewer names — signals stronger systems thinking
than a single polished, memorized answer ever could, because it demonstrates the reasoning generalizes
rather than having been looked up once for one project. That's also, not coincidentally, exactly the
signal a senior engineer is expected to carry into a brand-new codebase on day one.

## Talking points this chapter sets up

- "How do you know this is still current" is a hard, easy-to-defer design question in almost any
  system that touches evolving external data — not a sign any one of these nine projects was built
  carelessly.
- Distinguishing "is this instance technically valid/approved" from "is this the current version of
  the underlying thing" — the single most common conflation across all nine analogs.
- A concrete, reusable mechanism (group identity, supersedes pointer, get-current-version query) that
  answers the question for any of them, live.
- Naming the downstream system that must also learn about a new version, not just the record of the
  new version's existence.
