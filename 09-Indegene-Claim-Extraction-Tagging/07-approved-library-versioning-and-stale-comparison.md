# 07 — Approved-Library Versioning and Stale Comparison

## Why this chapter exists

Every chapter so far has treated the "approved-claims library" the Content Comparator checks
against (chapter 03) as a fixed, given thing — a list of approved sentences you compare new content
to. It isn't fixed. Legal/regulatory-approved source claims get **revised**: a label update changes
an efficacy percentage, a new safety signal forces a claim to be pulled entirely, a superseded study
gets replaced by a newer one with different wording. This chapter asks the question an interviewer
who understands the domain will eventually ask: **when the approved library changes, how do you stop
the Content Comparator from matching new content against a claim that's since been withdrawn — or
worse, leaving old, already-approved content sitting on the books as "supported" by a claim that no
longer exists?**

As with the rest of this course, there is no real source repository behind this chapter — everything
below is a plausible, technically detailed reconstruction of how a system like this would realistically
need to be built, not a verified account of Indegene's actual implementation. The goal is to reason
about the problem with the same rigor and specificity you'd use if you had read the source, while being
honest that this is a "here's how I'd build it, and here's the gap I'd expect" answer, not a "here's
what the code does" answer.

The honest answer has the same five-part shape as a well-handled "how do you handle X" question: what
a realistic implementation does today (and where the gap is), a manual workaround that closes most of
the gap without a schema change, a precise distinction between two concerns that are easy to conflate,
a genuine proposed design, and a forward-looking tie to the retraining pipeline in chapter 04.

## Part 1 — What a realistic implementation does today: the library is a flat, mutable list

Picture the approved-claims library the way chapter 03 describes it: a per-client table (or S3-backed
index) of approved claim text, each row roughly `{claim_id, claim_text, source_document_ref,
therapeutic_area}`, used to build the TF-IDF vectorizer's vocabulary and the sentence-embedding index
the Content Comparator searches at inference time. A realistic first-cut implementation of this table
has **no notion of time** in it at all — a row either exists (the claim is approved) or it doesn't
(someone deleted it or updated its text in place). When legal revises a claim — say, an efficacy figure
changes from "42%" to "38%" after a label update, or a claim gets pulled outright after a new safety
signal — the most direct, least-engineered way to reflect that is to **update the row in place** or
**delete it and insert a new one**. Either way, two things are true that matter a great deal:

1. **There is no link between the old wording and the new wording.** The system has no way to answer
   "what did this claim used to say" or "is this new claim a revision of that old one, or an entirely
   unrelated new claim." From the comparator's point of view, an update-in-place is indistinguishable
   from the claim having always said the new thing.
2. **Nothing automatically re-examines content that was already approved against the old wording.**
   If a piece of marketing content was compared against the approved library six months ago, matched
   claim `#4821` ("Drug X reduces symptom severity by 42%..."), and was approved and published on that
   basis, and claim `#4821` is *today* updated to say "38%" — that published content is now, silently,
   citing a superseded figure, and nothing in the pipeline knows to flag it. The comparator only ever
   runs at *review time*, against the library *as it exists at that moment*; it has no mechanism to
   re-run itself retroactively against every piece of content it already cleared.

This is the plain, first-sentence answer worth having ready: *"Today, in a realistic implementation of
this kind of system, the approved-claims library is basically a flat, mutable table — when legal revises
a claim, there's no built-in versioning that links the old wording to the new one, and there's no
automatic re-check of content that was already approved against the version that's now stale. The
comparator's job is to check new content against the library as it exists right now; it was never asked
to go back and re-examine what it already cleared."*

There's a second, subtler failure mode worth naming in the same breath: the comparator doesn't
necessarily query the live table row-by-row on every request — for performance, a realistic
implementation precomputes and caches the TF-IDF vectorizer's vocabulary and the sentence-embedding
vectors for the whole library (chapter 03's "cheap filter first, expensive check second" pattern only
works if the cheap filter's index is already built). If that cached index isn't rebuilt the moment the
underlying table changes — and a naive implementation would rebuild it on a schedule (nightly, say)
rather than on every write — there's a window where the comparator is matching against an index that
still contains a claim's *old* text even after the underlying table has already been updated, which is
a second, independent way stale comparisons happen.

## Part 2 — The manual workaround, achievable without a schema change

Nothing about this problem is unsolvable with the tables as they already exist — it's just entirely
process-driven rather than system-enforced, which is exactly the same shape of gap chapter 05 of the
Capco Document Uploader course describes for document revisions. A working, if manual, three-step
pattern:

1. **Legal flags a claim as withdrawn or revised**, through whatever internal process governs the
   approved-claims library (a regulatory affairs sign-off workflow, most realistically) — this step
   already has to exist independent of any engineering work, because legal owns the claims, not the
   ML pipeline.
2. **Someone runs a manual query**: "give me every piece of content whose Content Comparator result
   recorded a match against claim `#4821`." This is only possible at all if the comparator's output was
   ever persisted with **which specific approved claim it matched**, not just a pass/fail flag — if the
   review-result record only stored "supported: yes," this step requires re-running the comparator
   against the entire content archive from scratch, which is a materially more expensive workaround than
   a targeted lookup.
3. **Every content item that query returns gets manually re-queued for human review**, with a note that
   its supporting claim has changed, rather than being silently left in its original "approved" state.

The precise, defensible way to state the gap: *"the pieces needed to do this today already exist —
you can identify affected content and put it back in front of a reviewer — but it's entirely
coordination-by-convention. Nothing automatically triggers step 2 or step 3 when legal does step 1;
someone has to know a claim changed, know to go looking, and know which content to re-flag. And step 2
only works at all if the original match was recorded claim-by-claim, not as a bare boolean."* That last
clause is the detail that turns this from a vague "we don't automate that" into a specific, engineering-grounded
statement about exactly what data has to exist for the manual process to even be possible.

## Part 3 — Two concerns that are easy to conflate, and shouldn't be

This is the distinction most worth being crisp about under interview pressure, because it's genuinely
easy to blur:

- **"Textually/semantically similar to an approved claim"** is a pure similarity-search question — it
  asks whether some vector in whatever index the comparator currently has loaded is close enough to the
  new content's claim text. This is entirely what chapter 03's TF-IDF/edit-distance/sentence-embedding
  machinery answers, and it has **no innate concept of time or currency** — a vector is a vector,
  whether it was computed from a claim that's been approved for three years or one legal withdrew
  yesterday, unless something explicitly filters on status.
- **"Similar to a claim that is *currently* approved, not one that's been withdrawn"** is a completely
  different question — it requires knowing, at the moment of comparison, the *current status* of
  whichever claim the similarity search matched. Answering it needs a join against a claim's
  up-to-date status, not just a nearest-neighbor lookup in a similarity index.

Conflating these two is exactly how a withdrawn claim keeps quietly "supporting" new content: if the
comparator's similarity index still contains the withdrawn claim's vector (because the index hasn't
been rebuilt, or because the row was never actually removed, just marked withdrawn in a separate status
field the similarity search doesn't check), a new piece of content can score a high match against it,
get treated as "supported," and get auto-passed or approved — even though the thing it matched no
longer represents an approved company position. The correct mental model, worth stating explicitly:
**"matched something in the library" and "matched something the company can currently say" are two
different questions, and a comparator that only answers the first one is silently answering the second
one incorrectly whenever a claim has been revised or withdrawn.** This is the direct analog of
Part 3 in the Document Uploader course's versioning chapter, which draws the same kind of line between
"has this upload been approved" and "is this the current revision of a document" — different axes,
easy to blur, expensive to blur in a compliance context specifically because the cost of getting it
wrong here is a non-compliant claim reaching publication, not just a UX inconvenience.

## Part 4 — A genuine proposed design for real approved-claim versioning

This is explicitly a **proposal** — a plausible, defensible design for closing the gap, not a
description of anything confirmed to exist. It needs two things: a schema change to the approved-claims
library, and a corresponding change to what the Content Comparator persists about its own matches.

**The approved-claims library gains version and lifecycle fields:**

```python
# Proposed additions to the approved_claims table — illustrative, not confirmed as implemented
claim_family_id = Column(String(36), nullable=False)
# Stable identifier shared by every revision of "the same logical approved claim."
# Generated on first approval of a claim; carried forward by every later revision
# of that same claim's wording.

claim_version = Column(Integer, nullable=False, default=1)
# Monotonically increasing per claim_family_id. Revision 1 is the original wording;
# revision 2 is what legal approved after the label update; etc.

status = Column(String(20), nullable=False, default="ACTIVE")
# ACTIVE | WITHDRAWN | SUPERSEDED — SUPERSEDED means a newer version of the same
# claim_family_id exists and is now the current one; WITHDRAWN means legal pulled
# the claim entirely with no replacement.

superseded_by_claim_id = Column(String(36), nullable=True)
# The claim_id of the specific newer row that replaces this one, if any. Nullable
# because a brand-new claim family has no predecessor, and a WITHDRAWN claim (as
# opposed to a SUPERSEDED one) has no replacement at all.

status_changed_at = Column(DateTime, nullable=True)
# When this row left ACTIVE status — the timestamp that triggers Part 4's re-flagging.
```

"Get the currently approved wording of a claim family" becomes a defined query — *the row with a given
`claim_family_id` whose `status = 'ACTIVE'`* — instead of an ambiguous "whichever row happens to exist
right now" lookup. A claim's full revision history is just "every row sharing this `claim_family_id`,
ordered by `claim_version`."

**The Content Comparator's own output has to record the match at this same granularity, not as a bare
boolean.** Every stored comparison result needs `matched_claim_id` *and* `matched_claim_version` —
which specific version of which specific claim family this content was actually compared against and
approved on the strength of — plus the timestamp of the comparison itself. This is the piece that makes
Part 2's manual workaround unnecessary: with this stored, **withdrawing or superseding a claim can
directly and automatically fire a query** — "find every content-review result whose `matched_claim_id`
is this one and whose disposition was 'approved'" — and push each one back into the human review queue
with a clear reason ("supporting claim was revised on `status_changed_at`; original approval is now
based on stale wording"), rather than relying on someone remembering to go looking.

**The comparator's similarity search itself needs to be status-aware, not just index-aware.** The fix
for Part 3's conflation is to make the similarity index carry status as a first-class filter, not an
afterthought: maintain the "currently approved" index (used to decide whether *new* content is
supported) built only from `status = 'ACTIVE'` rows, and a separate, full historical index (used only
for audit — "did this ever match something approved, even something later withdrawn") built from every
row regardless of status. A match against the historical-only index that doesn't also appear in the
active index is precisely the signal that a piece of content is citing something no longer current — and
that distinction is only possible once status is modeled as data the similarity layer actually consumes,
not a field that lives in the table but never reaches the vectors.

## Part 5 — The forward-looking tie to the retraining pipeline (chapter 04)

This is the point worth volunteering to show the versioning problem doesn't stop at the comparator — it
reaches into the MLOps pipeline from chapter 04 too. Chapter 04's evaluation gate compares a freshly
retrained model against the currently-deployed one on a fixed held-out test set before promoting it.
That comparison is only meaningful if the **approved-claims library the new model (or the comparator
it's paired with) is evaluated against is the *current* one, not a stale cached snapshot** left over
from an earlier pipeline run. Concretely: if the retraining pipeline's evaluation stage pulls its
approved-library reference from an S3 snapshot that was taken before legal's most recent revision cycle,
the retrained model (or the comparator's rebuilt embedding index) gets validated against claims that no
longer reflect current legal reality — it could pass evaluation cleanly while being validated against
withdrawn wording, and nothing in the pipeline would surface that as a problem, because the evaluation
gate only checks "did accuracy improve against this reference set," not "is this reference set itself
current." A defensible fix: the data-validation stage at the front of the retraining pipeline (chapter
04's stage 1) should explicitly assert that the approved-library snapshot it's about to evaluate against
matches the library's current `status_changed_at` high-water mark — i.e., **fail the pipeline run outright**
if the reference snapshot predates the most recent claim revision, rather than silently training and
evaluating against stale ground truth. This is exactly the kind of failure mode that turns into a real
"bug found and fixed" story — chapter 08 picks it up directly as one of the four concrete narratives.

## The full interview-ready answer, assembled

> "A realistic implementation of the approved-claims library the Content Comparator checks against is,
> by default, a flat, mutable table — no versioning, no link between a claim's old wording and its
> revised wording. When legal revises or withdraws a claim, nothing automatically re-examines content
> that was already approved against the old version, and depending on how the comparator's similarity
> index is refreshed, there can even be a window where new content is still being matched against a
> claim's stale text after the underlying row has already changed. The workaround achievable without a
> schema change is manual: once legal flags a claim as withdrawn, someone queries which previously-approved
> content matched that specific claim and re-queues it for review — but that only works at all if the
> original match was recorded claim-by-claim, not as a bare pass/fail. It's also worth being precise
> that 'textually similar to an approved claim' and 'similar to a claim that's still currently approved'
> are two different questions — a similarity index has no innate notion of currency unless status is
> explicitly wired into it as a filter, and conflating the two is exactly how a withdrawn claim keeps
> quietly supporting new content. If I were building this properly, I'd add a `claim_family_id` and
> `claim_version`/`superseded_by_claim_id` to the approved-claims schema, record `matched_claim_id` and
> `matched_claim_version` on every comparator result instead of a boolean, and split the similarity index
> into an 'active-only' index that gates new content and a full historical index used only for audit —
> so withdrawing a claim can directly and automatically re-flag everything that was approved on the
> strength of it. And this reaches into the retraining pipeline too: an evaluation gate that validates a
> retrained model or a rebuilt comparator index against a stale cached snapshot of the approved library
> can pass cleanly while being validated against withdrawn wording, so the pipeline's data-validation
> stage needs to explicitly check that its reference snapshot is at least as current as the library's
> last revision before it runs — not just assume it is."

## Tying it back

The strongest part of this answer isn't the proposed schema in Part 4 — it's Parts 1 and 3: naming
plainly that a realistic version of this system has no built-in concept of a claim's currency, and
drawing a precise, correct line between "matched something in the library" and "matched something still
approved," so an interviewer can see those two ideas aren't being blurred under pressure. That precision
is exactly what a compliance-adjacent ML system needs from the person who built it — confidently
identifying the gap between "the model found a match" and "the match still means what it used to mean"
is a stronger signal than any specific schema field.
