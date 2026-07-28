# 06 — Document Reprocessing and Citation Deduplication

## Why this chapter exists

Course 5's document-uploader rebuild has a chapter answering a real interview question the
candidate was actually asked: "for revised versions of the same document, how are you handling
those?" That question generalizes past document-management CRUD systems — any pipeline that turns
a document into structured records has to answer some version of it. This CV pipeline is no
exception, and the honest, most likely follow-up an interviewer asks after hearing "5% to 85%
citation tracking accuracy" is some version of: **"a claim gets flagged during MLR review, someone
fixes the scan and resubmits it — does your pipeline know that's the same underlying document, or
does it just process it again and create a second set of citations?"**

**An important honesty note before going further, unlike course 5's chapter 05**: there is no real
source repository backing this course. Course 5 was rebuilt from actual proprietary code the
candidate had access to; this course was not. Everything below — including what "today's
implementation" is assumed to do — is a plausible, technically detailed reconstruction of how a
pipeline like this would realistically be built and what gap it would realistically have, based on
how the rest of this course describes the architecture (Chapters 01–05) and on how batch
CV/document pipelines are commonly built in production. Treat it as a well-reasoned, defensible
answer to a fair interview question, not a claim about verified internal Indegene implementation
details. The goal is to be *specific and confident* in the reasoning, while being explicit that the
specifics are illustrative.

The chapter has the same five-part shape as course 5's chapter 05: what a realistic implementation
does today, a manual workaround, a clean distinction between two concerns that are easy to conflate,
a genuine proposed fix, and the downstream tie-in that shows systems-level thinking.

## Part 1 — What a realistic implementation does today: reprocessing is not idempotent

Nothing described in Chapters 01–05 of this course — the OCR stage, the YOLO detector, the CNN
classifier, or the way they're composed — includes any check for "have I already produced a
structured citation output for this exact input before." Each stage is described purely in terms of
what it does to *one* input image; none of them reason about the input's relationship to prior
inputs. The most plausible real behavior, consistent with everything else this course says about the
production deployment (Chapter 00's architecture diagram, Chapter 04's batch-vs-real-time discussion):

1. A document lands in **S3** (a slide deck export, a scanned journal reprint, a leave-behind PDF
   flattened to images) under a client-scoped prefix.
2. An S3 event (directly, or via **Lambda + API Gateway** per Chapter 00) triggers a new pipeline
   job — a fresh **ECS Fargate** task pulls the document, runs Stage 1 (OCR) and orchestration, and
   invokes the **Sagemaker**-hosted YOLO and CNN models per document/page.
3. The job produces a structured citation output (`{claim_text, marker, bbox, linked_reference}`
   per Chapter 00) and writes it to S3, keyed by something derived from *this specific upload event*
   — a new job ID, a timestamp, or the incoming object key.

The direct, honest first sentence of the interview answer: *"As built, every object that lands in
that S3 prefix triggers a brand-new run of the full three-stage pipeline, keyed off that upload
event — there's no step that asks 'is this the same underlying document as something I already
processed,' so re-submitting a document produces a second, independent structured-citation output
sitting alongside the first, with nothing tying them together."* That's the same shape of honest
gap course 5 names for document versioning — not "we don't support that," but "the pieces to check
for it don't exist yet, and nothing stops a duplicate from being created."

This matters specifically because pharma content genuinely does get resubmitted. Plausible, concrete
triggers for this in an MLR-reviewed content pipeline: a scanned journal reprint gets rescanned at
higher DPI because the first pass was too blurry for OCR to read a footnote reliably; a slide deck
gets re-exported from PowerPoint after a formatting fix that has nothing to do with the citations on
it; an MLR reviewer flags one page as rotated or cropped wrong and asks for a corrected re-upload of
just that page. None of these are hypothetical edge cases for a pharma content pipeline — they are
the ordinary, expected lifecycle of a document under active review.

## Part 2 — The manual workaround, achievable today

As with course 5's uploader, nothing about this gap is unfixable *by hand* — it's just entirely
operator-driven rather than pipeline-enforced. A realistic manual workaround, using only what
Chapter 00's architecture already provides:

1. **Locate the prior run's output.** Structured citation output in S3 is plausibly organized by
   client prefix and source filename/timestamp (Chapter 00's per-client S3 isolation), so an
   operator (or a lightweight internal tool) can search that prefix for a prior output keyed to the
   same source document name or the same claim/slide identifier.
2. **Mark the prior output stale.** Since there's no `superseded` flag in the structured output
   schema itself, the realistic workaround is external to the pipeline — a tracking sheet, a status
   field in whatever content-management system routes documents to MLR review, or a manual S3
   tag/prefix move (e.g., into an `archived/` or `superseded/` sub-prefix) — so downstream
   consumers know not to trust the old output once the new one exists.
3. **Let the resubmission run through the pipeline as a new job.** The corrected scan gets processed
   completely fresh — full OCR, full YOLO detection, full CNN classification — exactly like any
   other document, since the pipeline has no lighter-weight "just check what changed" path.

This is a defensible, precise thing to say in an interview: *"the pipeline doesn't automate this,
but nothing prevents a human — an MLR coordinator, an ops engineer — from manually retiring the
stale output before or after the corrected one lands. The gap isn't that it's impossible to keep the
system consistent, it's that consistency depends entirely on someone remembering to do it, every
time, for every resubmission."* That's a stronger, more specific answer than either overclaiming
("we handle revisions automatically") or underclaiming ("no, we don't do anything about that").

**The real cost of this workaround, worth naming candidly:** reprocessing from scratch means paying
the full OCR + YOLO + CNN inference cost again for a document where, plausibly, 95% of the content
(and the citations on it) didn't actually change — only one page was rescanned, or only the
formatting shifted. That's a real, avoidable cost at the volume this pipeline runs at for two active
clients, independent of the correctness question Part 4 addresses.

## Part 3 — Two concerns that are easy to conflate, and shouldn't be

It's tempting to treat "the pipeline reprocesses everything from scratch every time" as a single
problem with a single fix. It isn't — there are genuinely two different situations hiding inside "a
document came in again," and conflating them leads to the wrong fix:

- **Concern A: this is genuinely a new document.** A different slide deck, a different journal
  reprint, a new leave-behind for a different drug/indication — content that has never been
  processed before and needs its own, independent set of citation detections. The pipeline's
  from-scratch processing is *correct* behavior here; there is no staleness problem to solve, and a
  system that's overly aggressive about treating "looks similar" as "is the same document" risks the
  opposite failure — silently skipping or merging genuinely distinct content because it happened to
  reuse a filename convention (e.g., two different clients' decks both named `final_v2.pptx`) or
  share a similar layout template.
- **Concern B: this is a corrected re-scan or re-export of a document already processed**, where the
  underlying claims and citations are the same but the input bytes differ — a rescan at higher
  resolution, a recompressed export, a deskewed/rotated correction, a re-flattened PDF after a
  cosmetic formatting fix. Here, from-scratch reprocessing isn't wrong in the sense of producing bad
  output, but it *is* wrong in the sense of producing a **second, independent, unlinked** set of
  citation detections for content that should have exactly one authoritative set.

The precise way to draw this boundary in an interview: *"the pipeline can't currently tell these
apart at all — both look identical to Stage 1: 'a new image arrived, go extract citations from it.'
The risk of getting this distinction wrong runs in both directions: if you treat every resubmission
as brand new, you get duplicate citation records for corrected re-scans (Concern B mishandled as
Concern A); if you get too aggressive about deduplicating on some proxy signal like filename or
approximate visual similarity, you risk silently dropping or merging genuinely new documents that
happen to look alike (Concern A mishandled as Concern B). The fix has to distinguish them correctly,
not just detect 'this looks similar to something I've seen.'"* This is the same shape of argument
course 5's chapter 05 makes about IWPB's approval state machine not being the same axis as document
versioning — naming precisely what a mechanism does and does not answer is the stronger signal, not
a vague "we handle that."

## Part 4 — A proposed design: content-hash dedup plus a `source_document_version` field

This is explicitly a **proposal** — a reasoned design for what a more complete fix would look like,
not a claim that it's implemented. It needs two complementary pieces: a cheap check that runs
*before* the expensive three-stage pipeline, and a field that ties every citation record back to
exactly what produced it.

**1. A content-hash dedup check ahead of the pipeline.** Two hashes, at two different tolerances,
because Part 3 established that "exact duplicate" and "corrected re-scan" need different handling:

```python
# Proposed — not implemented today. Runs once, cheaply, before Stage 1 (OCR) even starts.

def compute_exact_hash(image_bytes: bytes) -> str:
    """SHA-256 over the raw file bytes. Catches a byte-identical re-upload of the same
    file -- e.g., a document accidentally re-submitted through a content-management system,
    or a retry that landed twice. Two hits here mean literally nothing changed."""
    return hashlib.sha256(image_bytes).hexdigest()

def compute_perceptual_hash(image: "PIL.Image.Image") -> int:
    """A difference hash (dHash) over the decoded pixels. Tolerant of the kind of change a
    corrected re-scan actually introduces -- recompression, minor brightness/contrast
    correction, a few degrees of deskew -- while still being sensitive to genuinely different
    content. Returned as a 64-bit integer so two hashes can be compared with Hamming distance."""
    ...

def check_for_reprocessing(image_bytes, image, client_id, processed_index):
    exact_hash = compute_exact_hash(image_bytes)
    if (client_id, exact_hash) in processed_index:
        prior = processed_index[(client_id, exact_hash)]
        return {"action": "skip_exact_duplicate", "prior_job_id": prior["job_id"]}

    phash = compute_perceptual_hash(image)
    near_match = find_within_hamming_distance(phash, client_id, processed_index, threshold=6)
    if near_match is not None:
        return {"action": "supersede", "supersedes_job_id": near_match["job_id"], "phash": phash}

    return {"action": "process_new", "exact_hash": exact_hash, "phash": phash}
```

- **Exact hash match** (`skip_exact_duplicate`) means the bytes are identical to something already
  processed for this client — a true no-op re-upload (a retry, a double-submit). The pipeline should
  skip full reprocessing entirely and just return a pointer to the existing output — this is the
  cheapest, least ambiguous case, and the one safest to fully automate.
- **Perceptual hash match within a tolerance, but not an exact match** (`supersede`) is the Concern-B
  signal from Part 3 — likely the same logical document, re-scanned or re-exported. Rather than
  silently skipping it (which would risk missing a genuine content change that a human corrected the
  scan specifically to fix), the safer design still runs the full pipeline on the new input, but
  tags the new output with `supersedes_job_id` pointing at the prior run, and — critically — flips
  the prior run's citation records to a `superseded` status rather than leaving them live alongside
  the new ones. This mirrors course 5's `supersedes_document_id` proposal directly, adapted to a
  detection-output record instead of an uploaded-document record.
- **No match at all** (`process_new`) is Concern A — genuinely new content — and runs through the
  pipeline exactly as it does today, with its hash recorded in the index for future comparisons.

**2. A `source_document_version` field on every citation record**, tying the detection back to two
independent things that can each change on their own:

```python
# Proposed addition to the structured citation output schema described in Chapter 00
{
    "claim_text": "reduced onset by 40%",
    "marker": "1",
    "bbox": [206, 190, 212, 200],
    "linked_reference": "...",
    "source_document_version": {
        "content_hash": "a91f...",          # which exact input bytes produced this detection
        "supersedes_content_hash": None,     # or the prior version's hash, if this is a re-scan
        "yolo_model_version": "yolo-v5-superscript-2024-03",
        "cnn_model_version": "cnn-classifier-2024-06",
    },
}
```

This field answers two questions that are easy to conflate but need to stay separate: *"which
version of the source document produced this citation"* (the dedup/versioning concern this chapter
is about) and *"which version of the model produced this citation"* (a related but distinct concern
— Chapter 07 covers a real bug where a retrained model was redeployed without a version tag, making
it impossible to attribute a batch of results to the model that produced them). Both axes matter
independently: a citation set can be stale because the *source document* changed (this chapter) or
because the *model* that scored it changed (Chapter 07) — and a monitoring or reporting system that
only tracks one of the two can't tell which kind of drift it's looking at when accuracy shifts.
Tying both into one version field, rather than tracking them separately and hoping they stay in
sync, is deliberate: it means a query like "show me every citation that would need re-review because
either its source document or its scoring model has since changed" is a single well-defined lookup,
not a manual cross-reference between two disconnected systems.

**The honest limit of this proposal, worth stating unprompted:** a perceptual-hash threshold is a
tunable, imperfect heuristic, not a certainty — a document that's been *substantively* edited (a new
paragraph added, a citation actually changed) but happens to render at a similar overall visual
layout could, in principle, land inside the "near match" threshold and get incorrectly treated as
Concern B when it's actually Concern A. The safer default for a threshold that's uncertain is to
**always still run the full pipeline** on a `supersede` match (never skip processing based on
perceptual similarity alone, only skip on an *exact* byte hash) and let a human reviewer confirm the
supersede relationship before old citation records are actually retired — automating the detection
of a likely revision, but keeping a human in the loop for the consequential action of marking
something stale, exactly the kind of proportionate design course 5's chapter 06 argues for elsewhere
in this curriculum.

## Part 5 — The forward-looking tie to a downstream reporting/citation-tracking layer

This is the point worth volunteering, not waiting to be asked, because it's what separates "I'd add
a hash check" from "I understand this needs to propagate through the whole system." Chapter 00
describes this pipeline's output feeding **MLR compliance workflows** — a downstream layer that
almost certainly aggregates citation data for reporting: how many claims were reviewed this quarter,
which references are cited most often across a client's content library, which claims still lack a
correctly linked citation. **If reprocessing isn't idempotent, that reporting layer double-counts.**
A corrected re-scan that produces a second, unlinked citation set doesn't just leave a redundant
S3 object sitting around — it inflates every aggregate metric built on top of it: a "342 claims
reviewed this quarter" dashboard becomes 343 or 344 the moment a rescanned page runs back through
the pipeline, with no signal to the reporting layer that the two citation sets refer to the same
underlying claim. Worse, if the citation-tracking layer also drives a search or retrieval surface —
"find every place drug X's efficacy claim has been cited across our content library" — a stale,
superseded citation set sitting alongside the corrected one means that search can surface the *old*
claim wording or the *old* linked reference indefinitely, with no way for a downstream consumer to
know which of the two is current. Exactly as course 5's chapter 05 argues for a RAG index sitting
behind a document-management system, the fix here has to reach as far downstream as the citation
data itself travels — a `supersedes_job_id` field in this pipeline's own output is necessary but not
sufficient if nothing downstream ever reads or honors it.

## The full interview-ready answer, assembled

> "As built, this pipeline has no dedup or revision concept — every document that lands in S3
> triggers a brand-new run of the full OCR-YOLO-CNN pipeline, and a resubmission of the same
> underlying document, corrected or not, produces a second, independent set of citation detections
> with nothing linking it back to the first. The manual workaround today is entirely operator-driven
> — find the prior output, mark it stale in whatever system tracks MLR review status, let the
> resubmission process fresh — and it works, but it depends on someone remembering to do it every
> time. It's also worth being precise that there are two different things hiding inside 'the
> document came in again': a genuinely new document, where fresh processing is exactly correct, and a
> corrected re-scan of something already processed, where fresh processing is technically fine but
> produces an unwanted duplicate. If I were building a real fix, I'd add a cheap content-hash check
> ahead of the expensive three-stage pipeline — an exact byte hash to catch true duplicates outright,
> and a perceptual hash within a tolerance to flag likely re-scans as a 'probably supersedes an
> existing job' signal, with a human confirming before anything gets marked stale, since a similarity
> threshold can't be fully trusted on its own. I'd also tag every citation record with a
> `source_document_version` — which input hash and which model version produced it — since 'the
> source document changed' and 'the model that scored it changed' are two different kinds of drift
> that both need to be attributable, not conflated. And none of this matters only inside this
> pipeline — if there's an MLR reporting or citation-search layer downstream, which there almost
> certainly is, a duplicate detection set there means double-counted compliance metrics or a search
> result surfacing stale, superseded citation content — so the fix has to reach as far downstream as
> the citation data itself goes, not stop at this pipeline's own output."

## Tying It Back

The strongest part of this chapter isn't the proposed hash check in Part 4 — it's Parts 1 and 3:
stating plainly that reprocessing isn't idempotent today, and drawing a precise line between "a
genuinely new document" and "a corrected re-scan of one already processed" so an interviewer can see
the distinction is understood, not just gestured at. As with course 5's equivalent chapter, being
confident and specific about a real gap — and about exactly what a fix would and wouldn't solve — is
a stronger signal than either denying the gap exists or offering a vague "we'd add caching for
that." Chapter 07 picks up the other half of this concern directly: what happens when the *model*,
not the document, is the thing that changed underneath a batch of results.
