# 08 — Production Resilience and Operational Engineering

## Why this chapter exists

The same "production-grade, not just architecture-diagram-grade" chapter this curriculum builds into
every Indegene course: a realistic error-handling table for the batch screening pipeline, a
concurrency/scaling caveat specific to this project's bursty, self-hosted batch workload, four candid
bug-found-and-fixed narratives, and one honestly-named hardening gap.

**The same honesty note as the rest of this course**: there's no source repository behind this
platform, so every specific bug story and table row below is an illustrative, technically detailed
reconstruction, written to be as concrete as course 5's source-grounded equivalent chapter, not a
verified incident log.

## Error-handling table for the batch screening pipeline

| Failure | What should happen |
|---|---|
| A single PDF/abstract in the batch is malformed or corrupted (unreadable file, truncated download) | **Log and skip that document; do not halt the batch.** Route it to a `failed_ingestion` queue with the specific error, so the rest of the corpus keeps processing — one bad file should never block thousands of others (Chapter 4's Step Functions per-item error handling) |
| A document exceeds the model's usable context window | **Hard-block that document from silent truncation.** Flag it `needs_chunking_review` rather than either silently truncating (losing content a screening decision might depend on) or silently dropping it — see bug #1 below for what happens when this isn't enforced |
| SageMaker endpoint/batch job throttles or returns 5xx under load | **Bounded retry with backoff at the Step Functions state level, then dead-letter + alert.** Never a silent drop — a throttled screening call that's simply abandoned undercounts the corpus exactly like bug #1 |
| A screening decision is missing a `model_version` or `criteria_version` tag (Chapter 7) | **Hard-block at write time.** An untagged screening decision can never be reconciled when criteria or model versions change later — reject the write outright rather than accepting an unversioned record |
| A `needs_rescreen` reconciliation pass (Chapter 7) finds a decision whose criteria_version trails current | **Not a silent auto-fix.** Flagged for re-screening or human re-adjudication, never left as-is and never quietly overwritten without a record of the change |
| Extraction step produces structured output failing schema validation | **Hard-block; routed to a human-review queue**, never auto-accepted with defaults filled in — same discipline course 11's chapter 07 applies to SOC/ICF schema failures |
| CloudWatch metric/log emission fails | **Logged locally if possible, otherwise dropped. Never blocks the pipeline** — an observability failure should never become a screening-availability failure |
| Reviewer-queue notification fails to send | **Logged warning; the item still sits correctly in the queue** — delays awareness, not correctness |

The asymmetry worth naming if asked directly: **corpus-completeness and versioning-integrity failures
are hard, unconditional blocks** — a document silently dropped from the corpus, or a screening
decision silently missing its version tags, undermines the review's PRISMA-style accounting of the
full funnel and Chapter 7's ability to reconcile staleness at all. **Observability and notification
failures are logged and tolerated**, because their cost is a delay in someone finding out, not a gap
in the review's evidentiary record.

## Concurrency/scaling caveat: peak batch throughput vs. idle-time cost

The caveat specific to this project's shape, distinct from a steady-traffic chatbot's scaling story
(Chapter 4 introduces this; the operational engineering detail belongs here): provisioning enough
self-hosted SageMaker batch/async capacity for a review's **peak throughput need** — clearing a
kicked-off review's corpus within its target turnaround window — without over-provisioning for the
**idle time between reviews**, when demand for that same compute can sit near zero for days or weeks.

This is a materially different tuning problem from a steady-traffic service's autoscaling story. A
steady-traffic chatbot's load varies within a predictable, roughly repeating daily/weekly pattern
(course 11's chapter 03's business-hours-driven autoscaling), so target-tracking autoscaling on a
short time horizon works well. This project's load is **project-based and bursty**: a review kickoff
can demand a large amount of parallel batch throughput essentially immediately, then drop to
near-zero once that review's screening phase completes, with no guarantee the next kickoff is on any
predictable schedule. Autoscaling tuned for smooth, cyclical traffic (slow, conservative scale-out;
slower scale-in) fits this shape poorly — conservative scale-out directly lengthens a kicked-off
review's turnaround time, and slow scale-in means paying for idle GPU capacity across the (often much
longer) gaps between reviews. The tuning this project needs is close to the inverse: **aggressive,
fast scale-out on detected queue depth at kickoff, and comparatively fast scale-in once a batch's
queue clears** — accepting more scale churn in exchange for not paying for capacity with no
steady-traffic floor to justify holding it warm.

## Four bugs found and fixed

**1. A batch job silently dropped documents that exceeded the model's context window instead of
chunking or flagging them, undercounting the actual screened corpus.** The batch ingestion step's
context-window check was implemented as a filter, not a router — documents whose token count exceeded
the configured maximum were excluded from the batch entirely, with the exclusion logged at a verbosity
level nobody was routinely reviewing. The screening job reported "corpus fully screened" against a
count that silently excluded every oversized document, and the discrepancy only surfaced when a
reviewer, cross-checking the review's PRISMA flow diagram against the original search-result count
from the literature database, noticed the numbers didn't reconcile — a manual catch, not an automated
one. *What would have caught it earlier:* a **hard invariant check comparing the ingested-corpus count
against the original search-result count at the start of every screening run**, failing loudly on any
discrepancy instead of allowing a silently-filtered subset to be reported as "the corpus." This is
exactly the error-handling table's "hard-block, route to `needs_chunking_review`" behavior above,
retrofitted after the fact — the fix wasn't a smarter chunker, it was refusing to let an excluded
document disappear without a visible trace.

**2. A criteria-prompt bug was fixed mid-review without re-screening already-processed abstracts,
creating exactly the inconsistency Chapter 7 describes.** A prompt-template bug caused the model to
apply an inclusion criterion's date-range boundary as exclusive when it should have been inclusive
(excluding studies published exactly on the review's cutoff date). The bug was caught and fixed
quickly once identified — but the fix was deployed as a prompt-template patch without triggering a
reconciliation pass against abstracts already screened under the buggy version, because at the time
there was no `criteria_version`/`model_version` tagging mechanism distinguishing "screened under the
old prompt" from "screened under the fixed one." Studies published exactly on the cutoff date that had
already been screened (and wrongly excluded) before the fix stayed excluded; only abstracts screened
after the fix benefited from the correction — precisely the "some screened under old criteria, some
under new, no clear record of which" failure mode Chapter 7 names directly. *What would have caught it
earlier:* the dual-tagging and reconciliation design Chapter 7 proposes — had every screening decision
already carried a `criteria_version`/`model_version` tag at the time of the fix, the deployment would
have bumped the version identifier and `flag_stale_screening_decisions` would have automatically
surfaced every previously-excluded abstract near the cutoff date for re-screening, instead of leaving
the correction to apply only prospectively.

**3. A tokenizer/encoding mismatch on non-English abstracts silently produced garbage extractions
instead of erroring.** A subset of the corpus — non-English studies with English abstracts
machine-translated by the source database, a common pattern in multi-region reviews — contained
encoding artifacts (mis-decoded characters, mixed encodings from the translation pipeline) that the
tokenizer accepted without complaint, producing malformed token sequences the model then extracted
structured data from anyway. The result wasn't an error — it was a structurally valid but
semantically garbled extraction (a dosage value or population descriptor mangled into something
plausible-looking but wrong), a worse failure than a crash because a crash gets noticed and a
plausible-looking wrong answer doesn't automatically. It surfaced when a reviewer, spot-checking
extractions for a non-English subset, noticed values that didn't match the source abstract at all.
*What would have caught it earlier:* an **encoding-validation step at ingestion** — decoding every
document with an explicit, verified encoding and rejecting (routing to manual review) anything that
fails clean round-trip decoding, rather than letting a tokenizer silently accept malformed input.

**4. A batch job's per-instance concurrency cap wasn't enforced during a review-kickoff burst, and a
pile of near-simultaneous screening calls saturated a SageMaker instance's KV-cache memory budget,
causing a cascade of slow, then failing, invocations.** A kickoff fanned out several thousand
screening calls concurrently against a batch/async endpoint sized for *average* expected load, not the
instantaneous burst at kickoff. Without a per-instance in-flight-request cap acting as a bulkhead, the
endpoint accepted more concurrent requests than its KV-cache memory budget (Chapter 1's MLA
compression notwithstanding — even a compressed cache has a ceiling) could serve efficiently, and
latency for every in-flight call degraded together as the instance thrashed. *What would have caught
it earlier:* the same per-instance concurrency-cap bulkhead pattern course 11's chapter 07 applies to
its own SageMaker endpoint, sized specifically against this project's burstier kickoff profile
(Chapter 4's autoscaling caveat) rather than an averaged, steady-traffic assumption.

## A candidly-named hardening gap: the gold-standard evaluation set doesn't refresh automatically

Chapter 5's screening precision/recall evaluation depends on a gold-standard, human-adjudicated subset
staying representative of current screening practice. Today, that gold-standard set is built once, at
adapter-training time, and re-used across subsequent evaluation runs without an automatic trigger to
refresh it as a therapeutic area's typical literature (and typical edge cases) shift over time. That's
a genuine, worth-naming-candidly gap: an adapter can look like it's maintaining strong precision/recall
against a gold-standard set that has itself quietly drifted out of step with the kind of abstracts the
model is now actually screening, giving a false sense of continued quality. The fix is proportionate
rather than a rebuild: a periodic (e.g., quarterly, or triggered by a detected shift in the corpus's
topic distribution) refresh of the gold-standard set with newly human-adjudicated examples, reviewed
and approved through the same change-control discipline this platform already applies to prompt
templates and criteria changes — not an automatic, unreviewed swap, since the gold standard itself is
exactly the kind of artifact that shouldn't change silently either.

## Tying It Back

Production-grade for this pipeline means the same thing it means everywhere else in this curriculum:
known, bounded failure modes with a clear hard-block/log-and-continue asymmetry (corpus completeness
and versioning integrity never degrade to a warning; observability and notifications do), a scaling
posture tuned to this project's actual bursty, project-based traffic shape rather than a borrowed
steady-traffic assumption, four candid bugs — three of them undercounting/staleness/silent-corruption
failures rather than outright crashes, which is the pattern worth remembering: a self-hosted batch
LLM pipeline's most dangerous failures are usually the quiet ones — and one honestly-named gap in the
evaluation harness itself. As with the rest of this course, every specific here is an illustrative
reconstruction — the honest framing is "here's exactly how I'd expect this to fail, and exactly how
I'd guard against it," which is the right register for this material whether or not the specific
incident happened verbatim.
