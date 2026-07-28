# 99 — Interview Q&A: Systematic Literature Review Copilot

Read this last, after the chapters are fresh. Questions are grouped behavioral -> technical
deep-dive -> system design -> retrospective, roughly the order a real interview loop tends to move
through them, with a final section on the client/production framing this course shares with the rest
of the curriculum's Indegene courses.

**Q2, Q9, Q14, and Q17 are the questions most worth over-preparing for.** Q2 ("why self-host instead
of just using Bedrock/an enterprise API") is the first architecture question any interviewer will ask
about this project. Q9 is the explicit DeepSeek-vs-GPT-4-vs-Claude architecture comparison. Q14 is
model supply-chain/provenance verification. Q17 is model-version vs. criteria-version staleness. As
with the rest of this course, there's no source repository behind these answers — they're a
deliberately specific, technically grounded reconstruction, and saying so plainly when asked is part
of a strong answer, not a hedge that weakens it.

---

## Behavioral

### Q1. Tell me about this project — walk me through it end to end.

Use the STAR summary from `00-README.md` as the 90-second skeleton, then expand on any piece if
asked. Short version: "Pharma medical-affairs teams run systematic literature reviews — screening
thousands to tens of thousands of scientific abstracts against inclusion/exclusion criteria,
extracting structured data from the studies that pass, and synthesizing findings for regulatory
submissions and safety-signal work. That volume made it a genuinely labor-intensive process. I built
a batch screening pipeline — Step Functions and AWS Batch orchestrating calls to a self-hosted
DeepSeek model on SageMaker — that screened abstracts, extracted structured data from included
studies, and drafted a synthesis, with every output routed through a human medical-writer sign-off
queue before anything became part of the final SLR report. I chose to self-host an open-weight model
rather than call a managed API for three specific reasons: cost at this volume, fine-tuning control
over therapeutic-area reasoning, and having a fixed, versioned, auditable model artifact for a
process that ultimately feeds regulatory submissions."

### Q2. Why did you self-host an open-weight model instead of just using Amazon Bedrock or an enterprise LLM API? Isn't that a lot more operational burden for not much benefit?

This is the question this course is built to answer in depth (full treatment in Chapter 2); the
compressed version:

> "Three separate reasons, not one blurred argument. First, cost economics at this workload's actual
> volume — a single review can screen tens of thousands of abstracts, and a client running many
> reviews across many therapeutic areas accumulates that many times over in a year. Per-token managed
> API pricing scales linearly with that volume forever; self-hosted GPU capacity has a largely fixed
> cost that amortizes better the more sustained the throughput is. At this workload's sustained,
> high-volume shape, that math favors self-hosting.
>
> Second, fine-tuning control. Reliable screening depends on the model correctly applying a specific
> client's therapeutic-area vocabulary and a specific review's inclusion/exclusion criteria patterns
> — getting from 'usually right when prompted well' to 'reliably right, measured against a
> human-adjudicated gold standard' needs weight-level fine-tuning access, LoRA adapters trained per
> therapeutic area. A closed API constrains you to whatever fine-tuning surface the provider chooses
> to expose, if any.
>
> Third, and most specific to this domain: regulatory auditability. This process feeds regulatory
> submissions, which means it needs to be able to say precisely what model, what version, and what
> configuration produced a given screening decision. A managed API can be silently updated by its
> provider — the model identifier looks stable but the underlying weights or serving behavior can
> change without your initiation. A self-hosted, checksummed, versioned model artifact doesn't have
> that problem; it's a fixed thing you can point to indefinitely.
>
> Those three reasons are independent — even if a vendor offered fine-tuning tomorrow, the
> auditability argument would still hold, because they could still silently update the surrounding
> serving stack. That's why this wasn't a one-line 'open source is better' decision."

### Q3. What's the actual operational cost of choosing to self-host — what did you take on?

Be honest about the tradeoff rather than only selling the benefits (Chapter 2, Chapter 4, Chapter 8):
GPU instance provisioning and sizing, autoscaling tuned to this project's specific bursty,
project-based traffic shape (not a steady-traffic default), SageMaker endpoint and batch-job
monitoring, model-artifact provenance verification (Chapter 6) before every deployment, and owning
the retraining/evaluation loop for therapeutic-area adapters (Chapter 5) instead of relying on a
vendor's managed fine-tuning pipeline. None of that disappears just because a closed API would have
been simpler to stand up — the argument for self-hosting was never "it's less work," it was "the
specific reasons in Q2 outweigh the additional operational responsibility for this workload."

### Q4. Describe a time you had to justify a more complex architectural choice over a simpler one.

Frame it around the self-hosting decision itself, since it's the clearest example in this project: a
managed API call is architecturally simpler to stand up than a self-hosted SageMaker deployment with
its own fine-tuning, provenance, and staleness-tracking machinery. Justifying the more complex path
meant being explicit about which specific requirements the simpler path couldn't satisfy — not "self-
hosting is generally better," but the three concrete reasons in Q2, each traceable to a real
requirement (extreme volume, domain-specific fine-tuning, regulatory auditability) rather than a
preference.

### Q4b. Describe a time (real or expected) you had friction between an engineering team's automation goals and a domain expert's (e.g., a medical writer's) trust in the system. How would you handle it?

A strong answer names the friction honestly rather than glossing over it: "Medical writers and
regulatory-affairs reviewers are accountable for the SLR's final quality in a way an engineering team
isn't — they're the ones who'd have to answer for a missed study if a regulator asked. That naturally
makes them skeptical of a system that screens abstracts faster than any human could, and that
skepticism is appropriate, not a change-management problem to route around. The way to handle it isn't
to argue the model is trustworthy in the abstract — it's to make the specific failure modes reviewers
are implicitly worried about checkable: show the screening precision/recall numbers against a
gold-standard set they helped adjudicate (Chapter 5), show them the double-screening/adjudication
pattern (Q15b) that keeps a human genuinely in the loop rather than only spot-checking, and be
transparent that the system is deliberately tuned to favor over-including borderline abstracts rather
than risk a silent exclusion. Trust gets built by making the system's actual behavior inspectable
against the standard reviewers already use to judge each other's work, not by asserting the model is
good."

### Q5. How would you explain systematic literature reviews to someone with no pharma background?

A good, plain-language answer: "When a drug company needs to make a claim backed by 'the published
science,' they can't just cite a few favorite studies — they have to show they looked at *everything*
relevant, in a documented, repeatable way. That means searching the literature broadly, reading
thousands of abstracts against a fixed set of rules for what counts as relevant, keeping track of
every abstract that gets excluded and why, pulling structured data out of every study that makes the
cut, and writing up a synthesis of what the evidence says overall. It's the same rigor as a systematic
research survey in any academic field, except the stakes are that it might support a regulatory
submission or a safety decision, so the process itself has to be defensible, not just the conclusion."

---

## Technical deep-dive

### Q6. Walk me through Multi-head Latent Attention. What problem does it actually solve?

Full detail in Chapter 1; the interview-ready version: "Standard multi-head attention caches a
full-size key and value vector per attention head for every previous token — that KV cache grows with
sequence length and head count, and at long context and high concurrency it can become the dominant
consumer of a GPU's memory during inference, capping how many requests you can serve at once. MLA,
which DeepSeek-V2 introduced and DeepSeek-V3 continues to use, compresses the keys and values into a
single, much lower-dimensional latent vector before caching, and reconstructs the full per-head
keys/values from that compressed representation only when actually computing attention. So what's
stored per token in the cache shrinks from something proportional to `num_heads x head_dim` down to
the latent dimension, which is a direct, mechanical reduction in the KV cache's memory footprint — not
a vague optimization claim. That matters concretely for this project because it means more concurrent
abstracts can be screened per GPU instance for the same memory budget." The notebook
`01_multi_head_latent_attention_demo.ipynb` makes the memory-savings argument numerically on synthetic
data if asked to go one level deeper.

### Q7. Explain DeepSeek's Mixture-of-Experts design and its load-balancing approach.

"DeepSeek-V3 uses a fine-grained MoE design — many smaller experts rather than a few large ones, plus
some always-on shared experts, with only a small subset of the routed experts activated per token.
That gives the model a very large total parameter count, most of which lives across the experts, while
inference compute cost tracks the much smaller activated-parameter count for any given token — that's
the standard MoE efficiency story. The distinctive part is the load-balancing mechanism: instead of the
traditional approach of adding an auxiliary loss term that penalizes uneven expert utilization —
which works, but pulls the router's decisions away from pure quality-driven routing and can hurt model
quality if the balancing pressure is too strong — DeepSeek uses an auxiliary-loss-free approach: a
per-expert bias term that gets nudged up when an expert's been underused recently and down when it's
been overused, applied directly to routing scores rather than threaded through the training loss. That
decouples balancing the router from the model's main training objective, which DeepSeek's published
results report recovers quality a loss-based approach would trade away." Notebook
`02_moe_load_balancing_demo.ipynb` simulates the lopsided-vs-balanced utilization distributions this
produces if asked to demonstrate it concretely.

### Q8. Why does any of this — MLA, MoE routing — actually matter for a business use case? Isn't this just architecture trivia?

A strong answer connects it directly to this project's cost and throughput story, not just recites the
mechanism: "MLA's KV-cache compression is what lets a single GPU instance hold more concurrent
in-flight screening requests in memory at once, which directly increases batch screening throughput
per dollar of GPU instance-hour — that's Chapter 2's cost-at-volume argument made concrete at the
hardware level. And the MoE design's activated-vs-total-parameter split is what lets DeepSeek carry a
very large total parameter count — useful for the broad domain knowledge screening decisions draw on —
while keeping the actual per-token inference cost proportional to a much smaller activated subset.
Neither of these is trivia in this context; they're the specific mechanical reasons a self-hosted
DeepSeek deployment can serve this project's extreme-volume screening workload cost-effectively."

### Q9. How does DeepSeek's architecture compare to GPT-4's and Claude's?

The explicit comparison question, full depth in Chapter 3 (cross-referencing course 3's chapter 08 for
the GPT-4/Claude side):

> "The honest starting point is that the real axis separating these isn't capability, it's
> disclosure. DeepSeek's architecture — Multi-head Latent Attention for KV-cache compression, and a
> fine-grained Mixture-of-Experts design with a specific, published, auxiliary-loss-free
> load-balancing mechanism — is documented in DeepSeek's own technical reports; you can verify every
> claim about it against a public paper. GPT-4 and Claude are both proprietary, closed-weight,
> decoder-only Transformer models, and neither OpenAI nor Anthropic has disclosed comparable
> architectural detail. GPT-4 using MoE is widely reported and speculated via unofficial leaks, but
> never officially confirmed by OpenAI — I'd name that as a rumor, not repeat it as fact. Claude's
> architecture has essentially no comparable public speculation attached to it at all.
>
> The practical consequence that mattered most for this project: DeepSeek's weights are open and
> self-hostable, and GPT-4's and Claude's are not, at any price point — you cannot download or
> fine-tune either of their weights. That's the concrete reason this project's architecture — self-
> hosting, LoRA fine-tuning per therapeutic area, a checksummed and versioned model artifact for
> audit purposes — was only possible with a model like DeepSeek in the first place.
>
> And to be clear, none of that is a claim that DeepSeek beats GPT-4 or Claude on capability — it
> might not, on plenty of benchmarks. The case for DeepSeek here is architectural transparency, cost
> economics at this workload's volume, and fine-tuning control for a specific domain task, not
> unqualified superiority."

### Q10. Isn't "managed API access to an open-weight model via Bedrock" basically the same thing as self-hosting it?

No, and this is a distinction worth being precise about (Chapter 2, `00-README.md`): "Some open-weight
models, potentially including DeepSeek, show up in Bedrock's managed model catalog as an option. That
gives you a hosted API call, same as calling any other managed model — but you still don't control the
weights. You can't fine-tune at the weight level beyond whatever surface Bedrock chooses to expose, you
don't control exactly which checkpoint is being served or when it might be updated, and you don't have
a self-verified, checksummed artifact you can point to independently of the provider's own claims about
what's running. This project specifically chose to self-host on SageMaker instead — deploying the
actual downloaded, provenance-verified weights ourselves — for exactly the fine-tuning-control and
full-weight-audit reasons in Q2's third point. 'The model is open-weight' and 'we control the weights'
are different claims, and it's easy to blur them if you're not careful — the architecture only gets the
regulatory-auditability benefit if you're actually the one holding and verifying the weights."

### Q11. Why is most of this project's inference workload run as batch/async rather than real-time?

"Screening thousands to tens of thousands of abstracts is a large, well-defined, non-interactive batch
of work — nobody's waiting sub-second on any individual screening verdict the way a chatbot user waits
on a reply. That's exactly the shape SageMaker Batch Transform or async inference is built for: high
throughput and cost efficiency over low, predictable per-request latency. A real-time endpoint is still
the right choice for the smaller, genuinely interactive slice of this platform — a medical writer
refining a synthesis draft or re-checking a single abstract's screening decision — because that's a
human waiting synchronously. The design principle is matching the serving shape to the actual latency
requirement and volume profile of each part of the workload, not defaulting everything to whichever
pattern is more familiar." (Chapter 4 has the full sizing and autoscaling detail if pushed further.)

### Q12. Explain LoRA fine-tuning as applied to this project specifically.

"Course 11's platform used LoRA to adapt a base model's *generation format and register* for clinical
documents. This project uses the identical mechanism — a frozen pretrained weight matrix, two small
trainable low-rank matrices `A` and `B` such that the effective weight becomes `W + B·A`, with only
`A` and `B` trained — applied to a different target: *screening-decision reasoning*, conditioned on a
specific therapeutic area's vocabulary and a specific review's inclusion/exclusion criteria patterns.
Practically, that means separate LoRA adapters per therapeutic area, each trained on that area's
historical, human-adjudicated screening decisions, served off one shared frozen base model via
SageMaker's multi-adapter serving support — the same 'one base, many swappable adapters' pattern course
11 uses, just applied to a different axis of specialization."

### Q13. How do you evaluate whether the fine-tuned model's screening decisions are actually good?

"The standard, real SLR quality metric: precision and recall against a gold-standard, human-adjudicated
subset of abstracts (Chapter 5). Recall matters more here than in a lot of classification problems,
because the cost of a false exclusion — the model wrongly screening out a paper that should have been
included — is severe and largely unrecoverable: an excluded abstract never reaches full-text review or
extraction, so it can silently never make it into the review's evidence base at all. A false inclusion
is a recoverable cost — the human review gate downstream still catches it. So the practical design
choice is to bias the operating point toward higher recall even at some precision cost, and to
periodically spot-check a sample of the model's *excluded* abstracts specifically, since that's the
failure mode with no natural downstream catch otherwise."

### Q14. How do you know the DeepSeek weights you downloaded and deployed are genuine, and not a tampered or compromised checkpoint?

Full depth in Chapter 6; this is a real, non-hypothetical supply-chain question for any organization
self-hosting open-weight models, and it's worth naming that plainly before answering it:

> "A model checkpoint is a large binary blob of numeric weights — you can't eyeball it the way you
> might review downloaded source code, and a subtly altered checkpoint could behave identically to the
> genuine model on almost every input while differing on specific triggers, which functional testing
> alone might never surface. The mitigation is the same category of practice as verifying a downloaded
> software package against a publisher's signature: every model artifact is verified against a
> cryptographic hash DeepSeek publishes for that exact release before it's ever deployed to a SageMaker
> endpoint or batch job. A hash mismatch is a hard stop — the artifact is quarantined, not deployed, and
> the download source investigated, never silently accepted with a logged warning. That verification,
> along with the download source and timestamp, gets recorded permanently against the resulting
> model-version identifier, so months later you can trace any screening decision back to a specific,
> verified, genuine checkpoint. This isn't optional hardening for a project like this — it's the thing
> that makes the whole 'we know exactly what model produced this decision' auditability argument
> (Q2's third reason) actually backed by evidence instead of just asserted." Notebook
> `05_model_provenance_verification_demo.ipynb` implements exactly this hash-check pattern, including
> the tampered-checkpoint case being caught, if asked to demonstrate it concretely.

### Q15. What patient-data and licensing concerns come up in the source literature corpus, and how do you handle them?

"Two distinct concerns, not one (Chapter 6). Licensed literature-database content — a lot of an SLR's
corpus comes from client-licensed subscription databases, not just open sources like PubMed, and that
licensing carries real contractual restrictions on redistribution and cross-client use — so the
platform applies the same per-client S3/IAM isolation this curriculum's other Indegene courses use for
sensitive protocol content, so one client's licensed corpus and any fine-tuning data built from it
never leaks into another client's environment. Separately, published literature occasionally includes
patient-level detail — rare, but not zero — which gets the same conservative, human-reviewed handling
this curriculum applies to patient-facing content generally: flagged for reviewer attention, never
silently processed or used as fine-tuning material without the same scrutiny any patient-level data
would receive under the client's own policies."

### Q15b. Real SLR methodology often uses double-screening — two independent screeners, with disagreements adjudicated by a third. Where does the model fit into that pattern?

A thoughtful answer treats the model as a participant in that pattern rather than a replacement for
it: "The cleanest fit is to have the model act as one of the two 'screeners,' with a human reviewer as
the other, rather than having the model screen alone and a human only spot-check afterward. Every
abstract gets an independent model verdict and an independent human verdict; where they agree, that's
a strong signal the decision is reliable; where they disagree, it routes to adjudication — in this
case, a senior reviewer, the same role a third human screener plays in an all-human double-screening
setup. That has two benefits over 'model screens everything, human reviews some of it': it keeps a
human genuinely independent judgment in the loop for every abstract rather than only for a sampled
subset, and the agreement/disagreement rate itself becomes a live, ongoing signal for Chapter 5's
precision/recall tracking — a rising disagreement rate for a given therapeutic area is an early
warning that the adapter for that area may need retraining, well before a scheduled gold-standard
evaluation would catch the same drift."

---

## System design

### Q16. Design the end-to-end batch screening pipeline for this project.

Structure the answer around the architecture in `00-README.md` and Chapter 4:

- **Ingestion**: literature corpus (PubMed, client-licensed sources) lands in S3, per-client
  bucket/prefix isolated, with the original search-result count recorded as an invariant to check the
  ingested corpus against later (Chapter 8, bug #1).
- **Orchestration**: Step Functions defines the pipeline as an explicit state machine — screen, wait/
  aggregate, extract (gated on screening's included set), synthesize (gated on extraction completing)
  — with per-state bounded retry so one bad document or one throttled call never halts the batch.
  AWS Batch manages the actual parallel compute-job scheduling within each stage.
- **Inference**: the bulk of calls (screening, extraction across thousands of abstracts) hit the
  self-hosted DeepSeek model via SageMaker Batch Transform/async inference, sized for throughput; a
  smaller real-time endpoint serves the interactive review/summary-refinement slice.
- **Versioning**: every screening decision is tagged with both a `model_version` and a
  `criteria_version` identifier at write time (Chapter 7), enabling reconciliation whenever either
  changes.
- **Human review**: every synthesized output and every flagged/reconciled screening decision routes to
  a medical-writer sign-off queue — nothing reaches the final SLR report artifact unreviewed.
- **Security/observability**: VPC/Private Endpoints, per-client IAM scoping, Secrets Manager,
  CloudWatch across every stage — the same backbone the rest of this curriculum's Indegene courses use.

### Q17. A review's inclusion/exclusion criteria get refined halfway through screening. Walk me through exactly what should happen.

This is the staleness question this course is built to answer (Chapter 7 in full); the compressed,
system-design-framed version:

> "First, the key thing to get right conceptually: there are two independent things that can go stale
> here, not one — whether the model version screening abstracts is current, and whether the criteria
> version a given screening decision was made against is current. They drift on completely different
> timelines and must be tracked separately, because a review can have a perfectly current model while
> still carrying thousands of decisions made against superseded criteria wording, or the reverse.
>
> Concretely: every screening decision is tagged with both a `model_version` and a `criteria_version`
> identifier when it's made. When criteria change, a reconciliation pass runs — `flag_stale_screening_
> decisions` — that finds every decision whose `criteria_version` trails the review's new current
> version, and flags it `needs_rescreen`. Depending on volume, those get either automatically
> re-screened against the current criteria and current model, or routed to a human reviewer for direct
> re-adjudication. Critically, a re-screened decision goes through the exact same human-confirmation
> gate a first-time decision would — a criteria update is never a fast path around review, because a
> criteria refinement is very often exactly the moment a decision most needs a careful second look, not
> the moment to auto-accept it.
>
> The failure this prevents is a review that silently mixes two different screening standards across
> its corpus with no record of which abstract got which — which breaks the whole premise of a
> PRISMA-style review being one coherent, documented process."

### Q18. How would this system scale if the client tripled the number of concurrent reviews it runs?

Name the actual bottlenecks and how each is addressed, in the style of course 11's equivalent scaling
question: (1) **batch inference throughput** — self-hosted DeepSeek serving via Batch Transform/async
with MLA's KV-cache efficiency (Chapter 1) scales better per GPU-dollar than provisioning separate
capacity per review, but autoscaling policy has to handle more simultaneous kickoff bursts rather than
one at a time (Chapter 4, Chapter 8's concurrency caveat); (2) **reviewer capacity** — the real
constraint, since human medical-writer review doesn't scale the way compute does; the screening
precision/recall tuning from Q13 (bias toward recall, triage flagged items) matters more at scale to
keep reviewer attention on what actually needs it; (3) **reconciliation-pass volume** — tripling
concurrent reviews means tripling the chance of overlapping criteria refinements landing near the same
time, which makes Chapter 7's `flag_stale_screening_decisions` mechanism load-bearing rather than an
edge-case nicety, the same way course 11's chapter 07 names a version-race concern becoming more
pressing at higher trial volume; (4) **per-client isolation overhead** — more concurrent clients means
more separately-isolated S3/IAM/adapter boundaries to provision and monitor, which is a real
operational cost that scales roughly linearly with client count, not something that gets cheaper with
scale.

---

## Retrospective

### Q19. What would you change if you rebuilt this today?

A strong answer names concrete, plausible improvements rather than "nothing":

- **Dual-version tagging from day one** (Chapter 7) rather than retrofitting it — the bug in Chapter
  8's story #2 (a criteria fix deployed without triggering reconciliation) happens specifically because
  the tagging and reconciliation machinery didn't exist yet when the fix shipped; building it in from
  the start is much cheaper than bolting it onto a platform already treating "current criteria" as
  implicit.
- **A refreshing, not static, gold-standard evaluation set** (Chapter 8's named hardening gap) — a
  gold-standard set that doesn't itself get periodically refreshed with new human-adjudicated examples
  can give a false sense of continued screening quality even as the underlying corpus's typical content
  shifts.
- **Encoding validation at ingestion from the start** (Chapter 8, bug #3) rather than discovering it
  from a reviewer's spot-check — a corpus this multi-lingual should assume malformed encodings will
  show up, not treat them as a surprise.
- **A hard invariant check comparing ingested-corpus count against original search-result count on
  every screening run** (Chapter 8, bug #1) — cheap to build, and it's the single check that would have
  caught the silently-dropped-oversized-documents bug immediately instead of via a reviewer's manual
  PRISMA-diagram reconciliation.
- **Multi-adapter serving designed in from the start** (Chapter 5), rather than treating per-
  therapeutic-area adapters as a later optimization — designing the serving layer around swappable
  adapters from day one avoids re-architecting deployment once the number of covered therapeutic areas
  grows.

Close by naming what would *not* change: the human medical-writer sign-off gate on every synthesis and
every reconciled screening decision. Every improvement above makes that reviewer's job faster and
better-triaged — none of them remove the reviewer, because that's the one design decision a
regulatory-submission-supporting process should never compromise on.

### Q20. What's the single biggest risk in this system if you had to name one?

A defensible answer: the recall-vs-precision tuning in Q13 — because a false exclusion has no natural
downstream catch, the single biggest risk isn't a visible failure at all, it's a *quiet* one: a
relevant study that the model wrongly excludes and that nobody ever looks at again, silently
under-representing the review's evidence base. That's precisely why periodic spot-checking of excluded
abstracts (Q13) and biasing the operating point toward recall are treated as load-bearing design
decisions rather than nice-to-haves — the system's failure modes that matter most here are the ones
that don't announce themselves.

---

## Client & production framing

### Q21. This is framed around the same pharma clients as the rest of the Indegene courses in this curriculum — what does that isolation look like here specifically?

Following the same pattern courses 7, 9, 10, and 11 establish: per-client S3 bucket/prefix isolation
for the literature corpus and extracted-data artifacts, per-client IAM role scoping so a compromised or
misconfigured credential on one client's pipeline has no path to another client's data, and per-client
(or namespace-isolated) LoRA adapters (Chapter 5) so one client's fine-tuning signal never mixes into
another's model artifact — same rationale as course 11's isolation story for two competing pharma
sponsors: their literature-review scope, criteria, and any competitively sensitive research
questions are exactly the kind of thing that must never be addressable across client boundaries.

### Q22. Why does this project sit on AWS specifically, using SageMaker, rather than a different cloud?

Consistency with the rest of this curriculum's Indegene courses' established AWS production
conventions (root README's Client & Production Context) — S3 for the corpus and artifacts, SageMaker
for self-hosted model inference (both the batch/async and real-time paths, Chapter 4), Step
Functions/AWS Batch for pipeline orchestration, CloudWatch for observability, Secrets Manager for
credentials, all behind the same VPC/Private-Endpoint security boundary courses 7–11 use. That
consistency isn't incidental — it means the operational discipline (per-client isolation, IAM scoping,
observability conventions) an Indegene team already has for its other pharma platforms transfers
directly to this one, rather than requiring a parallel set of practices for a different cloud vendor.

### Q23. If you had to describe the single most distinctive thing about this project relative to the rest of the Indegene courses in this curriculum, what would it be?

Volume and workload shape. Courses 7, 9, and 10 process documents at real but comparatively bounded
per-document volume; course 8 is conversational, human-paced; course 11 generates a handful of derived
documents per protocol amendment. This project's defining characteristic is that a *single* review can
mean screening tens of thousands of abstracts, which is what makes batch/async inference the primary
serving path rather than an afterthought (Chapter 4), what makes cost-at-volume a first-order
architectural argument rather than a footnote (Chapter 2), and what makes a quiet, undercounting
failure (Chapter 8's bugs) a more dangerous failure mode than a loud one — at this volume, a small
per-document error rate compounds into a real gap in the evidence base if nothing is watching for it.

A good closing line for this question, if there's time for one: the model choice (Chapter 1, Chapter
3) and the deployment shape (Chapter 4) both trace back to that same volume fact rather than being
independent preferences — DeepSeek's KV-cache-efficient attention mechanism and cost-favorable
self-hosting economics only become the *right* call because the workload is large and sustained
enough to need them; the same architecture would be over-engineering for a review screening a few
hundred abstracts. Being able to name that connection — that the scale of the problem is what
justifies the scale of the solution, not the other way around — is usually the strongest signal an
interviewer is listening for in a systems-design-flavored follow-up to this question.
