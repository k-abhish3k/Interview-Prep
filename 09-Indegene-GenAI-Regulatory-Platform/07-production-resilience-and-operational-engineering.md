# 07 — Production Resilience and Operational Engineering

## Why this chapter exists

This is the "production grade, not just architecture-diagram grade" chapter — a realistic
error-handling table for the generation pipeline (which failures must block a document from
reaching a reviewer versus which are safe to just log), a genuine concurrency caveat around
protocol amendments, four candid "bug found and fixed" narratives specific to a clinical-document
generation platform, concrete timeout/retry values for the Sagemaker LLaMA 3 endpoint, and one
named hardening gap.

The same honesty note from chapter 06 applies here with equal force: there is no source repository
behind this course, so every specific number, bug story, and table row below is an **illustrative,
technically detailed reconstruction** — built to be as concrete and confident as course 3's
source-grounded equivalent chapter, but not a verified account of a real incident log. Treat it as
"here's exactly how I'd expect this to fail, and exactly how I'd guard against it," which is the
right register for this material in an interview regardless of whether the specific bug happened
verbatim.

## The realistic error-handling table for the generation pipeline

The central design question this table answers: **which failures must hard-block a document from
ever reaching a reviewer, versus which are safe to log and move on from?** Getting this asymmetry
right is the actual substance of "production-grade" in a patient-safety domain — a system that
blocks on everything is unusable, and a system that blocks on nothing is dangerous.

| Failure | What should happen |
|---|---|
| Factual-grounding-check flags an unsupported quantity or claim (chapter 05's grounding check) | **Hard-block.** The section is forced into `FLAGGED_FOR_REVIEW`, never `CLEAN_DRAFT_FOR_REVIEW` — this must never be a soft warning a busy reviewer can miss in a sea of "clean" items; it changes which queue the item lands in |
| SOC/PLPS/ICF structured output fails schema validation (malformed JSON, missing required key) | **Hard-block.** Routed to `needs_review` per chapter 05's `generate_soc_entry` pattern — never silently retried into an auto-accepted state, and never "best-effort parsed" with defaults filled in |
| A newly generated document has a missing or null `protocol_version` | **Hard-block at write time** — reject document creation outright. An unversioned document is strictly worse than no document at all, since chapter 06's staleness check has nothing to compare it against and it can never be flagged `needs_regeneration` if it later goes stale |
| SOC entry's `diffs_against_version` does not equal `protocol_version - 1` | **Hard-block.** Reject the SOC entry before it reaches a reviewer — a version-skipping SOC (chapter 06, Part 4) could silently fold two amendments into one diff and bury an intermediate change |
| Section/heading detection finds zero recognizable headings on a document expected to follow a known Protocol template | **Hard-block for the whole document.** Route to manual structuring rather than auto-processing on ambiguous, unlabeled chunks — a document this platform can't confidently segment is a document it shouldn't guess at generating from |
| Sagemaker endpoint invocation times out or returns 5xx/throttling | **Bounded retry with backoff, then dead-letter + alert** — never a silent drop; see timeout values below |
| Country-comparison graph database query times out or errors | **Logged and retried; comparison view marked "temporarily unavailable."** Not a hard block on the underlying documents themselves — the reviewer still has the full source Protocol and country variant text to fall back on manually, so a graph outage degrades a convenience layer, not the ground truth |
| CloudWatch metric or structured-log emission fails | **Logged locally if possible, otherwise dropped.** Never blocks the pipeline — an observability failure should never become a patient-facing availability failure |
| Reviewer-queue notification (email/dashboard alert) fails to send | **Logged warning; the item still sits correctly in the queue.** A notification failure delays *awareness* of a pending review, not the *correctness* of the review state itself |

The asymmetry worth stating explicitly if asked "how does this pipeline handle failure": **content-
integrity and versioning failures are hard, unconditional blocks — grounding, schema, and
`protocol_version` invariants never degrade to a warning, because the cost of a false negative here
is a patient-facing or regulatory document being wrong.** Observability and notification failures
are logged and tolerated, because the cost of a false negative there is a delay in someone finding
out, not an incorrect document reaching use. That's the same shape of asymmetry course 3's chapter
06 names between IWPB and non-IWPB failure handling — the newer, higher-stakes path is deliberately
the more conservative one.

## Concurrency caveat: two amendments to the same protocol, processed concurrently

A realistic failure mode worth naming directly: **two amendments to the same Protocol, submitted
or reprocessed close together in time, racing on which one "wins" as the current version.** This
isn't far-fetched — a regulatory-affairs coordinator resubmitting a corrected upload minutes after
an initial one, or a batch reprocessing job overlapping with a live upload, both produce two
concurrent attempts to determine "what is `protocol_version` for this Protocol right now."

The naive implementation — read the current max version, add one, write it back — is a classic
check-then-act race:

```python
# The naive, race-prone version — illustrative of the bug, not the fix
current = db.query("SELECT MAX(protocol_version) FROM protocols WHERE protocol_id = ?", pid)
new_version = current + 1
db.execute("INSERT INTO protocols (protocol_id, protocol_version, ...) VALUES (?, ?, ...)", pid, new_version)
```

If two amendment-processing requests both read `current = 2` before either has written its `INSERT`,
both compute `new_version = 3` and both succeed — two Protocol rows both claiming to be version 3
for the same protocol, with no defined ordering between them. The practical damage isn't silent
data loss (both amendment documents are preserved) — it's **ambiguity about what "current" means**,
which is exactly the ground truth chapter 06's whole design leans on: the `needs_regeneration` flag,
the SOC `diffs_against_version == protocol_version - 1` invariant, and every reviewer-facing
staleness queue all assume there is one unambiguous "current version." Two rows both claiming to be
v3 breaks that assumption at the root.

**The real fix**: assign `protocol_version` through a mechanism the database enforces atomically,
not one the application code merely intends to enforce — a `UNIQUE (protocol_id, protocol_version)`
constraint plus an `INSERT ... ON CONFLICT` retry loop (or an actual DB sequence/auto-increment
scoped per `protocol_id`), so the *second* concurrent writer's insert fails fast on the constraint
and is forced to re-read the new current version and retry as v4, rather than two writers each
independently believing they're v3. This is the same category of fix as course 3's advice for its
maintenance-loop race (a DB-enforced single source of truth beats an application-level
check-then-act, every time concurrency is a real possibility rather than a theoretical one) — the
lesson generalizes cleanly across both courses.

## Four bugs found and fixed

**1. A section-boundary chunking bug split a dosing-instruction paragraph mid-sentence, and the
generated ICF omitted a safety caveat that lived in the second half.** Chapter 01's chunker is
designed to split at paragraph boundaries, never mid-sentence — but a Protocol section with an
unusually long single paragraph (a dosing rule immediately followed, in the same paragraph, by its
safety-monitoring caveat — "...administer 100 mg every 4 weeks. Discontinue immediately if injection-
site reaction Grade 2 or higher occurs.") exceeded the chunker's `max_chars` threshold *mid-
paragraph*, and the fallback path split it at the nearest whitespace instead of refusing to split at
all. The generation call for the first chunk produced a plausible-looking ICF dosing paragraph with
no visibility into the second chunk's safety caveat, and — because nothing in the grounding check is
designed to detect an *omission* (it checks whether generated claims are supported by the source, not
whether the source's claims all made it into the output) — the missing caveat sailed through
grounding cleanly and reached the reviewer queue as a "clean" draft. This is a genuine, candidly-
worth-naming near miss: a caveat this material getting silently dropped is exactly the kind of error
a reviewer, skimming a "clean" draft with some trust in the pipeline, could plausibly miss. *What
would have caught it earlier:* a **golden-document regression test suite** — a small, curated set of
real (or realistic synthetic) Protocol sections with known, hand-verified expected chunk boundaries
and a list of sentences/caveats that must never be split apart or dropped, run on every change to the
chunking logic. Grounding checks catch fabrication; they don't catch omission — that has to be a
separate, explicit test category.

**2. A version-tracking bug let a reviewer approve a PLPS section while looking at a cached
rendering of the OLD Protocol version.** The review-queue UI's rendering layer cached a generated
section's HTML/preview by `section_id` alone, to avoid re-rendering identical content on every page
view. When Protocol v2 triggered regeneration of a PLPS section that had previously been generated
(and cached) under v1, the regeneration produced new content correctly and wrote it to the database
— but the rendering cache, keyed only on `section_id` and not on `protocol_version`, kept serving the
stale v1 rendering to the reviewer's browser. The reviewer approved what they were shown, believing
it was the new v2 draft, when the underlying database record (and audit log) correctly reflected v2
content the reviewer never actually saw rendered. This is precisely the Concern-B failure chapter 06
warns about — a document that is *supposed* to reflect the new version doing so in the database while
a caching layer silently defeats it at the point a human actually looks. *What would have caught it
earlier:* **cache invalidation tied explicitly to `protocol_version`, not just `section_id`** — the
cache key must be `(section_id, protocol_version)`, so a version bump structurally cannot serve a
stale rendering, rather than relying on someone remembering to bust the cache on every regeneration
path.

**3. A graph-diff bug in Country Specific Protocol Comparison missed a downstream requirement change
because the traversal only checked direct edges, not transitive ones.** Chapter 04's `REFERENCES`
traversal was implemented to check exactly one hop out from a changed requirement — it correctly
found "does anything directly reference the requirement Germany just modified," but a real Protocol
graph can have chains: requirement A `REFERENCES` B, and B `REFERENCES` C. If Germany's variant
modifies C, the one-hop implementation correctly flags B (which directly references C) but misses A
entirely, even though A's meaning depends on C transitively through B. This surfaced when a reviewer,
manually reading a section the pipeline hadn't flagged, noticed it referenced a requirement that had,
two hops away, actually changed for that country — a reviewer catch, not an automated one, which is
exactly the kind of near-miss worth naming candidly rather than only citing the successes. *What
would have caught it earlier:* **graph-traversal test cases that specifically cover chains of two or
more `REFERENCES` edges**, not just single-hop fixtures — a test suite built entirely from
one-hop examples will pass even when the traversal function silently stops at depth one, because
nothing in a one-hop-only fixture set can distinguish "correctly traverses one hop" from "correctly
traverses to a fixed point." Notebook `06_transitive_graph_diff_fix.ipynb` (chapter 07's companion
notebook, see below) builds exactly this bug and its fix side by side.

**4. A `protocol_version` race (the concurrency caveat above, before the DB constraint fix was
added) caused a SOC entry to diff against the wrong "previous" version.** Two near-simultaneous
amendment-processing jobs for the same Protocol both read `current_version = 4` and both wrote a row
claiming `protocol_version = 5` before the unique-constraint fix existed. The SOC-generation job that
happened to run against the *second* of those two v5 rows computed its diff against the section text
of `protocol_version = 4` — correct by the letter of "diff against the immediately-prior version" —
but by the time a reviewer opened it, the *other* v5 row (from the concurrent writer) had also been
processed, and the platform had, briefly, two different "current version 5" documents with two
different SOC entries in circulation, until the ambiguity was manually reconciled. *What would have
caught it earlier:* the same fix named above — a DB-enforced `UNIQUE (protocol_id, protocol_version)`
constraint would have made the second writer's insert fail immediately and force a retry as v6,
rather than allowing two rows to both claim v5. This bug is the concrete "and here's what actually
goes wrong if you don't fix it" companion to the abstract concurrency caveat earlier in this chapter.

The common thread worth stating as the takeaway across all four: **three of the four are omission or
staleness bugs, not fabrication bugs** — the grounding-check machinery in chapter 05 is well-suited
to catching a model inventing something; it is not, by itself, suited to catching a caveat that got
silently dropped (bug 1), a reviewer looking at stale content through no fault of the generation
pipeline (bug 2), or a comparison that stopped one hop short of where it needed to go (bug 3). A
mature test strategy for this kind of platform needs a distinct category of "did everything that
should be present survive the pipeline," not only "is everything present actually supported by the
source" — those are different questions, and testing only the second gives a false sense of
completeness.

## Concrete timeout, retry, and scaling values for the Sagemaker LLaMA 3 endpoint

Illustrative, plausible values for a platform running sustained daily generation traffic against a
fine-tuned LLaMA 3 real-time endpoint (chapter 03), worth having ready as specifics rather than
vague gestures at "we set reasonable timeouts":

- **`SAGEMAKER_INVOKE_TIMEOUT_SECONDS = 60`** per section-generation call. A full ICF or PLPS
  section at realistic output length can legitimately take tens of seconds on a quantized model
  under concurrent load — a timeout set too aggressively (e.g., 10-15s) would trigger spurious
  retries on requests that were simply still working, adding load rather than relieving it; 60s
  gives real generation calls headroom while still bounding a genuinely hung request.
- **Bounded retry: up to 2 retries, backoff 2s then 6s, on retryable errors only** (timeout, `500`,
  `503`, throttling) — never retried on a `4xx` (malformed request/input), since a bad request will
  simply fail identically on retry and a retry loop there just wastes an invocation slot that a
  legitimate request could have used.
- **A per-instance concurrency cap (e.g., 4 in-flight invocations per GPU instance)** acting as a
  simple bulkhead — without one, a burst of concurrent section-generation calls (several sections of
  the same ICF regenerating together after an amendment, chapter 06) can saturate a single instance's
  batching capacity, causing every in-flight call's *actual* latency to creep toward the timeout
  simultaneously — a cascading-timeout failure mode that a naive "just increase the timeout" fix
  would make worse, not better, since it would let more concurrent load pile onto the same
  instance before anything fails.
- **Auto-scaling target-tracking on invocations-per-instance (or GPU utilization), min 1 / max N
  instances**, with a **longer scale-in cooldown than scale-out cooldown** — matching chapter 03's
  observation that usage is sustained during business hours and near-zero overnight; scaling out
  quickly when regulatory-affairs teams start their workday but scaling in cautiously avoids
  thrashing instances up and down across short lulls within a working day.
- **A shorter, separate timeout for the grounding-check call (e.g., 10s)** where the grounding check
  is itself a lighter second model call (chapter 05's defense #4) rather than the simple string-
  matching version — it's a narrower, cheaper task than full section generation and shouldn't share
  the same generous budget, since a slow grounding check delaying the reviewer queue is a worse
  trade-off than a slow generation call.

## A candidly-named hardening gap: the grounding-check watchlist is hardcoded

Notebook `04_protocol_comparison_pipeline.ipynb`'s `CLAIM_WATCHLIST` — the set of strong,
unsupportable outcome-claim words the grounding check flags (`cure`, `guarantee`, `eliminate`,
`completely`, `harmless`, and similar) — is a literal, hardcoded Python `set` in source. That's a
genuine, worth-naming-candidly hardening gap rather than something to gloss over: as the platform
generates content across more therapeutic areas, more sponsors, and more languages, the vocabulary of
unsupportable claims worth watching for will need to grow and vary (an oncology trial's problematic
language isn't identical to a dermatology trial's), and today that requires a code change and
redeploy, not a configuration update a clinical-content or regulatory-affairs specialist could make
directly. The fix is straightforward and low-risk, mirroring course 3's session-secret fix in spirit
if not in specifics: move the watchlist out of source and into an externally editable store (a config
table, a versioned data file loaded at startup, or a small admin surface for regulatory-affairs
specialists to propose additions) — reviewed and approved through the same kind of change-control
process this platform already applies to prompt templates, so watchlist growth stays deliberate and
auditable rather than becoming an unreviewed free-for-all, while no longer requiring an engineering
redeploy for every new term.

## Tying It Back

Production-grade for a clinical-document platform means the same thing chapter 06 said it meant for
versioning: failure modes are known, bounded, and the team can say precisely what blocks a document
from a reviewer versus what merely gets logged, what happens when two amendments race, and what's
already been caught and fixed versus what's still a named, open gap. This chapter's profile: hard,
unconditional blocks on every content-integrity and versioning invariant (grounding, schema,
`protocol_version` presence, SOC version-skipping), a real concurrency caveat with a concrete DB-level
fix, four candid bugs — three of them omission/staleness failures the grounding-check machinery alone
was never designed to catch, which is itself the most important lesson in this chapter — concrete
Sagemaker timeout and scaling numbers grounded in an actual daily-usage traffic shape, and one
honestly-named hardening gap (the hardcoded claim watchlist) with a proportionate fix rather than a
rebuild. As with chapter 06, every specific here is illustrative reconstruction, not verified fact —
and saying so plainly, while still being this specific, is the stronger interview answer in a
patient-safety domain than either vague hand-waving or overclaiming certainty about an unverified
implementation.
