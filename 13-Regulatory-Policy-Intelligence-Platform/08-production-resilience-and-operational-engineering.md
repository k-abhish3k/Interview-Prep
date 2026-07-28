# 08 — Production Resilience and Operational Engineering

## Why this chapter exists

The rest of this course covers what makes this platform *work* — architecture, fine-tuning, governance.
This chapter is the layer underneath, matching the pattern every other course in this curriculum
carries: a realistic error-handling table, a concurrency/scaling caveat specific to this course's own
architecture, several candid bug-found-and-fixed narratives, and one hardening gap named plainly rather
than glossed over. **The same honesty constraint applies here as everywhere else in this course past
Chapter 1**: illustrative, plausible, technically grounded reconstruction, not a verified incident log.

## A realistic error-handling table for the self-hosted serving stack

| Failure | What happens | Why |
|---|---|---|
| GPU out-of-memory (OOM) on a large batch | The serving layer (vLLM/TGI or Azure ML's inference server) rejects or queues the oversized batch rather than crashing the whole process; the orchestration service catches the rejection and retries with a smaller batch size / splits the request | A single oversized batch crashing the entire serving process would take down every in-flight request on that replica, not just the one that was too large |
| Endpoint autoscaling lag during a volume spike | Incoming requests queue at the orchestration layer with a bounded queue and backpressure (reject/defer new submissions past a queue-depth threshold) rather than piling up unboundedly while new GPU replicas spin up | GPU instance provisioning is not instant (Chapter 4) — the system has to survive the gap between "load increased" and "capacity increased" without silently dropping or corrupting in-flight work |
| A malformed document breaks the extraction pipeline | That document is isolated, logged with the specific parsing/extraction failure, and routed to a manual-review queue; the batch of documents around it continues processing | Mirrors the general "isolate one failure from the batch" pattern used across this candidate's other document-processing pipelines (course 7, course 9) — one bad PDF shouldn't stall an entire day's ingestion run |
| Mixtral endpoint returns a low-confidence or malformed structured-output response (fails schema validation) | Routed to a fallback: a stricter, schema-constrained re-prompt attempt once, then — if still malformed — flagged for mandatory human review rather than silently passed downstream | A malformed extraction silently reaching a downstream compliance workflow is worse than an honest "this needs a human to look at it" |
| Azure AI Search retrieval call (hybrid retrieval layer) fails or times out | Classification/extraction proceeds using the document's own text alone, with the output explicitly flagged as "produced without retrieval-augmented context" | Mirrors course 1's retrieval-outage handling — degrade gracefully rather than fail the whole pipeline for a dependency that isn't strictly required to produce *some* output |
| Model registry unreachable at inference time (Chapter 6/7 provenance lookup fails) | The endpoint continues serving the already-loaded model artifact (no restart-on-registry-failure), but the request is tagged as "provenance unverified for this call" in the audit log | Serving on an already-loaded, already-verified-at-load-time artifact is safe; treating a transient registry blip as a reason to stop production entirely is not proportionate — but the audit trail has to be honest about the gap |
| A fine-tuned adapter regresses on the domain evaluation set post-deploy | Automatic rollback to the previous registry-pinned adapter version; the regression is logged and blocks further promotion until investigated | Never let a regression discovered *after* promotion keep serving just because rolling back is inconvenient — the whole point of registry-pinned artifacts (Chapter 6, Chapter 7) is that rollback is a one-line operation, not a redeploy-from-scratch |

Two design philosophies worth naming explicitly, matching the pattern established across this
curriculum:

- **User/reviewer-facing messages stay generic and actionable; logs stay detailed.** A compliance
  reviewer sees "this document needs manual classification — automated extraction was inconclusive,"
  never a raw GPU OOM stack trace or a schema-validation exception.
- **A capacity or dependency degradation gets a graceful fallback; a data-integrity risk gets a hard
  stop.** Retrieval outages and registry blips degrade gracefully because a slightly worse answer is
  better than no answer. A hash-mismatched model artifact (Chapter 6) or a malformed structured-output
  response gets a hard stop, because serving on either risks silently wrong output flowing into a
  regulated compliance workflow.

## The concurrency/scaling caveat specific to self-hosted GPU serving

This is the caveat worth naming unprompted if asked "how does this scale under load," because it's a
genuinely different failure shape from every other Capco course in this curriculum: **unlike a managed
API with effectively unlimited backend capacity from the caller's perspective, a self-hosted endpoint
has a hard capacity ceiling set by however many GPU instances have actually been provisioned.**

Course 1's chatbot, calling Azure OpenAI, can absorb a traffic spike by leaning on Azure OpenAI's own
enormous backend fleet — the caller's problem under a spike is quota (TPM/RPM limits), not the
provider running out of physical compute. This platform's self-hosted Mixtral endpoint has no such
elastic backstop: if a volume spike genuinely exceeds the GPU capacity currently provisioned, and
autoscaling hasn't caught up yet (Chapter 4's autoscaling-lag scenario, above), requests queue, then
back up, then start hitting the bounded-queue backpressure limit — in a way a managed API, at this
platform's volume, simply wouldn't, because the provider has vastly more spare capacity sitting behind
the API than this platform would ever provision for itself. This is a genuine, structural tradeoff
against the cost argument in Chapter 2, worth stating plainly rather than glossing over: self-hosting
trades away the managed API's effectively-infinite elastic capacity for a lower steady-state cost, and
that trade only stays favorable if provisioned capacity and autoscaling responsiveness are sized
against realistic peak volume, not just average volume. The mitigation is the bounded queue plus
backpressure design above (fail predictably and visibly rather than degrade silently under overload),
paired with capacity planning that explicitly budgets for known volume patterns (regulatory filing
deadlines cluster around known dates, similar to course 11's protocol-amendment-deadline burst pattern)
rather than sizing only for average daily volume.

## Bugs found and fixed, specific to self-hosted LLM serving

Illustrative, plausible, technically detailed bug narratives — the kind of concrete story a candid
engineer can describe from a project like this, not a claim of a specific verified incident.

**1. A quantization configuration that silently degraded extraction accuracy on one document category.**
Moving the serving layer to an 8-bit quantized configuration (Chapter 4) passed general-purpose
evaluation cleanly — aggregate classification accuracy on the domain evaluation set (Chapter 5) barely
moved. What the aggregate number hid: accuracy on one specific, less-common taxonomy category — filings
involving unusually dense cross-referenced clause structures — dropped meaningfully, because that
category's classification depended on subtler distinctions the quantized model handled less reliably
than the full-precision one, while the far more common, more templated categories were essentially
unaffected. This was caught only because the domain evaluation set from Chapter 5 was scored
per-category, not just in aggregate — an aggregate-only evaluation would have shipped this
configuration to production. *What would have caught it earlier, systematically:* per-category
regression scoring as a hard gate on any serving-configuration change, not just any model-weight
change — quantization is a configuration change to how the same weights are computed, and it deserves
the same evaluation rigor as a new adapter version, not a lighter check.

**2. A tokenizer mismatch between the fine-tuning pipeline and the serving stack.** The fine-tuning
pipeline (Chapter 5) and the production serving stack (Chapter 4) each independently loaded Mixtral's
tokenizer — the fine-tuning side from a locally cached copy, the serving side from a copy bundled with
the serving framework's container image. A version skew between the two (the serving container had been
rebuilt against a slightly newer packaged tokenizer revision than the one the most recent fine-tuning
run had used) meant a handful of rare characters common in certain regulatory filings' formatting
(section-symbol and other legal-document-specific glyphs) tokenized differently between training and
serving — the adapter had learned patterns conditioned on one tokenization, and was being served
against a subtly different one. The visible symptom was narrow and easy to miss: slightly degraded
extraction accuracy specifically on documents containing those characters, nothing that looked like an
obvious error. *What would have caught it earlier:* pinning and hash-verifying the tokenizer artifact
itself, alongside the model weights, in the same provenance record Chapter 6 describes for the base
model — treating the tokenizer as part of the versioned artifact, not an incidental dependency assumed
to always match.

**3. An autoscaling policy that scaled on CPU utilization instead of GPU utilization.** An early version
of the AKS-based serving deployment (Chapter 4) inherited a generic autoscaling policy template from an
unrelated CPU-bound service elsewhere in the client's Azure estate, scaling replica count on CPU
utilization. Under real inference load, GPU utilization was consistently saturated while CPU
utilization on the same nodes stayed low — the orchestration and request-handling overhead is genuinely
CPU-light relative to the GPU-bound inference work itself — so the autoscaler never triggered a scale-out
even as request queues grew and latency degraded, because the metric it was watching wasn't the actual
bottleneck. *What would have caught it earlier:* load-testing the autoscaling policy itself, not just
the serving stack's raw throughput — specifically verifying that a synthetic sustained-load test
actually triggers a scale-out event, not just that the model serves requests correctly at a single fixed
replica count.

**4. A schema-constrained extraction call that silently truncated on documents near the context-window
boundary.** The orchestration layer (Chapter 4/5) chunked long regulatory filings to fit comfortably
within Mixtral's effective context window, but the chunking boundary calculation didn't account for the
token overhead of the structured-output schema instructions themselves (the JSON-schema prompt
scaffolding for extraction, per Chapter 5's output format). For documents whose content chunk landed
right at the token budget boundary, the combined prompt occasionally exceeded the window, and the
serving layer's own truncation behavior silently dropped the tail of the schema instructions rather than
the input document — producing extraction output that looked structurally plausible but was missing one
or more expected fields, with no error raised. *What would have caught it earlier:* a boundary-condition
test suite specifically targeting documents sized to land at or near the token-budget edge, rather than
only testing with typical-length documents comfortably inside the window — the same category of
off-by-one boundary bug course 1's Chapter 7 describes for its own truncation logic, recurring here in a
different pipeline.

The common thread, worth stating as the takeaway: **all four are "the pipeline runs without error, so it
looks fine" bugs** — nothing threw an exception, no error-rate dashboard fired, and in three of the four
cases the degradation was narrow enough (one category, one character class, one boundary condition) that
an aggregate or happy-path test would have missed it entirely. That's exactly why per-category domain
evaluation (Chapter 5), full artifact provenance including the tokenizer (Chapter 6), and boundary-
condition-specific testing all earn their keep here — a self-hosted serving stack has more independently-
versioned moving parts (base weights, adapter, tokenizer, quantization config, serving framework,
autoscaling policy) than a single managed API call ever exposes, and each one is a place a narrow, quiet
regression can hide.

## A hardening gap, named candidly: no automated tokenizer-version check at serving startup

The tokenizer-mismatch bug above (Bug 2) was fixed for the specific incident, but the underlying gap
wasn't fully closed: the serving stack's startup sequence verifies the **base model weights hash**
against the registry (Chapter 6) before accepting traffic, but does **not** independently verify the
tokenizer artifact's version/hash against what the currently-loaded adapter was actually fine-tuned
against. That means a *future* skew of the same shape — a serving-container rebuild that happens to pick
up a different packaged tokenizer revision — would not be caught automatically at deploy time; it would
again have to be caught the hard way, through a quiet accuracy regression on a narrow slice of documents,
exactly like the first time.

The fix is proportionate and already scoped, not a large undertaking: extend the startup provenance
check (which already exists for the base model weights) to also hash-verify the tokenizer artifact
against the tokenizer hash recorded in the adapter's own registry entry, and hard-fail startup — not
silently proceed — on a mismatch. This is a natural, small extension of infrastructure that already
exists (Chapter 6's hash-verification pattern) rather than a new capability that has to be built from
scratch, which is exactly why it's worth naming as a known, scoped gap rather than something requiring a
larger redesign.

## Tying It Back

Production-grade for a self-hosted LLM serving stack isn't "the model classifies documents correctly" —
it's "every failure mode is known and bounded, the one genuinely different scaling risk versus a
managed-API architecture (a hard GPU capacity ceiling, not elastic API backend capacity) is named
explicitly rather than assumed away, the bugs that are easy to ship here are the quiet, narrow,
'looks fine, isn't' kind that only per-category evaluation and full artifact provenance catch, and the
one remaining hardening gap — no automated tokenizer-version check at startup — is named candidly with
a scoped, already-understood fix, not glossed over as if the platform were already complete.
