# 04 — Deployment Architecture: AWS SageMaker

## Two serving shapes, and why this workload is mostly the "boring" one

SageMaker offers a few distinct ways to serve a model, and picking the right one per workload matters
more than defaulting to whichever one is most familiar. The two relevant here:

- **Real-time inference endpoints** — an always-on (or auto-scaled-to-zero-with-cold-start, depending
  on configuration) HTTPS endpoint that answers individual requests with low, predictable latency.
  This is the shape course 7, 9, and 10's real-time Sagemaker endpoints use, and the shape course 11's
  fine-tuned LLaMA 3 endpoint uses for interactive regulatory-affairs section generation.
- **Batch Transform (or async inference)** — submit a large batch of inputs (or a queue of requests
  for async inference) and get results back once processing completes, without an interactive
  request/response loop holding a connection open per call. Designed for high-throughput, non-
  interactive, cost-efficient bulk processing rather than sub-second responsiveness.

The workload-shape question this project actually needs to answer is which of those two matches
**most of the volume** in a systematic literature review. Screening thousands to tens of thousands of
abstracts against inclusion/exclusion criteria is not a conversation — nobody is sitting at a screen
waiting sub-second for each individual screening verdict, the way a chatbot user waits for each
turn. It's a large, well-defined, non-interactive batch of work that needs to complete within a
reasonable turnaround (hours, not milliseconds) at the best achievable throughput and cost. That is
precisely the workload shape **Batch Transform / async inference** is built for, and it's why this
project treats batch/async as the **primary** serving path for the bulk of its volume, with a
real-time endpoint reserved for a smaller, genuinely interactive slice of the work.

## Where a real-time endpoint still earns its place

Not every part of this platform is bulk screening. The **interactive review and summary-drafting
phase** — a medical writer refining a synthesis draft, asking a follow-up question about why a
specific abstract was excluded, or re-running a single abstract's screening decision after a criteria
clarification — is a small-volume, latency-sensitive, human-paced workload, structurally identical to
the chatbot-style interaction pattern course 1 and course 8 are built around. That slice belongs on a
**real-time SageMaker endpoint**, sized for responsiveness rather than throughput, because the
person on the other end of that call is waiting for it synchronously.

The design principle worth stating explicitly, because it's the kind of thing a system-design
interview question is fishing for: **match the serving shape to the workload's actual latency
requirement and volume profile, not to whichever pattern is easiest to stand up first.** Most of this
platform's inference volume (bulk screening, extraction across thousands of abstracts) has no
sub-second latency requirement and a strong cost-per-unit-of-work requirement — batch/async is the
right default. A small slice (interactive review, summary refinement) has a real latency requirement
and low volume — a real-time endpoint is the right, and only sensible, choice there. Defaulting
everything to a real-time endpoint because it's the more familiar pattern would mean paying for
always-on, latency-optimized capacity to serve a workload that never actually needed sub-second
responses — the wrong tool for the dominant share of the work.

## GPU instance sizing considerations

Sizing a GPU instance family for self-hosted DeepSeek inference (whether behind the real-time
endpoint or the batch/async path) comes down to a few generic but load-bearing considerations, kept
deliberately generic here since exact instance types and memory figures shift with both DeepSeek
model-variant choice and AWS's current GPU instance catalog:

- **Model weight footprint at the chosen precision/quantization.** A full-precision deployment of a
  large DeepSeek variant needs substantially more GPU memory than a quantized deployment of the same
  model — the same accuracy/cost tradeoff course 11's chapter 03 walks through for LLaMA 3 applies
  here: a more aggressively quantized model for high-volume, well-templated screening calls, and a
  less-quantized model reserved for the more nuanced synthesis/extraction calls where subtlety of
  reasoning matters more.
- **KV cache headroom at expected concurrency.** Chapter 1's MLA discussion is directly load-bearing
  here — because DeepSeek's KV cache footprint per token is compressed relative to a standard
  multi-head-attention model, a given instance can hold more concurrent in-flight sequences' caches
  in the same GPU memory budget, which is a direct, quantifiable throughput advantage for the batch
  screening workload this project runs at volume.
- **Batch/async worker parallelism vs. real-time endpoint instance count.** For the batch path, sizing
  is really about total throughput needed to clear a review's corpus within its target turnaround
  window (more, potentially smaller, parallel workers processing chunks of the corpus concurrently);
  for the real-time path, sizing is about serving the interactive slice's much lower, spikier request
  volume at acceptable per-request latency — a different sizing exercise with a different objective
  function, and conflating the two is a common mistake worth naming explicitly if asked.

## Autoscaling for a bursty, project-based batch workload

The batch screening workload's traffic shape is fundamentally different from a steady-traffic
chatbot's, and that difference should drive the autoscaling design, not just the instance choice.
Reviews start and finish — a review kicking off means a burst of demand for many abstracts to be
screened as fast as reasonably possible; between reviews (or between a review's screening phase and
its later extraction/synthesis phases), demand for that stage's compute can drop close to zero. That's
a **project-based, bursty** load profile, not the "sustained business-hours traffic" shape course
11's chapter 03 describes for its interactive regulatory-affairs endpoint.

The right autoscaling posture for this shape: scale batch/async worker capacity aggressively on
review-kickoff (favor fast scale-out over cost-conservatism when a review's corpus lands and needs to
clear its screening queue), and scale back down promptly once a batch completes rather than holding
capacity warm for an uncertain next arrival — the inverse cooldown asymmetry from course 11's
interactive-endpoint pattern, where scale-in was deliberately *slower* than scale-out to avoid
thrashing across short lulls within a business day. Here, the lulls between reviews are the norm, not
the exception, so paying for idle capacity between projects is the cost failure mode most worth
guarding against, while under-provisioning at kickoff (leaving a review's screening queue backed up
for longer than necessary) is the throughput failure mode worth guarding against on the other side.
Getting that balance right — enough headroom to clear a kicked-off review's queue promptly, without
paying for GPU capacity sitting idle for the days or weeks between review kickoffs — is the specific
cost/latency tradeoff this workload's bursty, project-based shape creates, distinct from a steady-
traffic service's simpler "scale with request rate" autoscaling story.

## Orchestrating the multi-stage pipeline: Step Functions / AWS Batch

The screen -> extract -> synthesize pipeline (`00-README.md`'s architecture diagram) is a genuine
multi-stage workflow with real dependencies between stages — extraction only runs on abstracts that
passed screening, synthesis only runs once a review's included-study set is stable — which is exactly
the shape **AWS Step Functions** (as a state-machine orchestrator) and **AWS Batch** (as the
compute-job scheduler for the actual bulk-processing work within each stage) are built to coordinate:

- **Step Functions** defines the pipeline as an explicit state machine: a screening state that fans
  out across a review's corpus (invoking the batch/async SageMaker path per chunk of abstracts),
  a wait/aggregate step once screening completes, an extraction state that runs only against the
  screening stage's included set, and a synthesis state gated on extraction completing. Step
  Functions' built-in retry/catch semantics per state are what implement the bounded-retry,
  don't-halt-the-whole-batch-on-one-failure behavior Chapter 8's error-handling table specifies.
- **AWS Batch** manages the actual compute-job queueing and scaling for the bulk work within a stage
  — submitting many parallel screening jobs against chunks of the corpus, letting AWS Batch's job
  queue and compute-environment autoscaling handle fanning that work out across available capacity
  without the orchestration layer itself needing to manage worker pools directly.
- Together, this separates **workflow logic** (what stage runs after what, what triggers a retry,
  what routes to the human review queue) from **bulk compute scheduling** (how many parallel workers
  process a stage's volume right now) — a cleaner separation than trying to hand-roll both
  responsibilities inside a single orchestration script.

## The same security boundary as the rest of this curriculum's Indegene courses

Consistent with courses 7, 9, 10, and 11: the SageMaker endpoint(s), the S3 buckets holding the
literature corpus and extracted-data artifacts, and the Step Functions/Batch orchestration all sit
inside a **VPC**, reachable only via **Private Endpoints** — no component in this pipeline is directly
reachable from the public internet. **IAM roles** scope exactly what each component (the orchestration
layer, the SageMaker endpoint, the S3 buckets) can access, **Secrets Manager** holds any credentials
the pipeline needs (external literature-database API credentials, for instance), and **CloudWatch**
provides end-to-end observability across the Step Functions executions, the Batch jobs, and the
SageMaker endpoint/batch-transform-job metrics — the same monitoring backbone course 7's chapter
covers for its own multi-stage AWS pipeline.

## Tying It Back

The deployment-architecture answer worth having ready: this platform's dominant workload — screening
and extracting across a large, non-interactive literature corpus — runs on **SageMaker Batch
Transform / async inference**, because throughput and cost efficiency matter far more than sub-second
latency for that volume, while a smaller **real-time SageMaker endpoint** serves the genuinely
interactive review/summary-drafting slice where a human is waiting synchronously. GPU instance sizing
leans on Chapter 1's MLA-driven KV-cache-memory advantage to fit more concurrent screening work per
instance; autoscaling is tuned for a bursty, project-based traffic shape (aggressive scale-out at
review kickoff, prompt scale-in between reviews) rather than a steady-traffic profile; and Step
Functions plus AWS Batch split workflow orchestration from bulk compute scheduling across the
screen -> extract -> synthesize pipeline. All of it sits behind the same VPC/Private-Endpoint/IAM
security boundary this curriculum's other Indegene courses establish — a self-hosted model doesn't
get a pass on that discipline just because it isn't a third-party API call.
