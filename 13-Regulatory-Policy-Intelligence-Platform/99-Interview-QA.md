# 99 — Interview Q&A: Regulatory & Policy Document Intelligence Platform

Read this last, once the chapters and notebooks are fresh. Q1 opens with the behavioral summary — your
answer skeleton for "tell me about this project." **Q2 is the near-certain opening technical question**
the moment an interviewer hears "self-hosted open-weight model at a bank" — it's placed right up front
deliberately. **Q3 (the architecture comparison to GPT-4/Claude), Q4 (model supply-chain/provenance),
and Q5 (taxonomy staleness)** are featured prominently right after it, because each is a genuinely
distinctive angle this project can own that most candidates describing a GenAI project can't speak to
with the same precision.

---

### 1. (Behavioral) Tell me about this project.

Use the STAR summary from `00-README.md`, practiced to under 90 seconds:

> A Capco banking client's compliance and legal teams were manually triaging a very high volume of
> regulatory filings and internal policy documents against the bank's own taxonomy — far more volume
> than the bank's existing GenAI projects handled, at a scale where per-token API pricing would have
> been prohibitively expensive, and without the weight-level transparency the client's model-risk
> function wanted for audit purposes. I was asked to build a document intelligence platform that could
> classify, extract, and summarize this content, fine-tuned to the client's proprietary taxonomy, fully
> inside the bank's existing Azure security boundary. I evaluated a managed API — including Azure AI
> Foundry's own hosted Mixtral offering — against self-hosting the same open-weight model, and chose to
> self-host Mixtral 8x7B on Azure ML managed online endpoints, GPU-backed and autoscaling, behind the
> same VNet/Private Endpoint/Azure AD boundary as the rest of the client's Azure estate. I built a
> hybrid retrieval layer over the policy corpus, fine-tuned Mixtral via LoRA on the client's taxonomy,
> and designed a model registry tracking both the base-model weights hash for supply-chain provenance
> and the taxonomy version each adapter was trained against, so the two could never drift silently
> against each other. *(Illustrative)* The cost model showed self-hosting becoming cheaper than
> continued API billing well within the platform's expected first-year volume, while giving the
> model-risk function a fully inspectable, hash-verified artifact it could evaluate independently.

---

### 2. Why did you self-host an open-weight model instead of just using Azure OpenAI, or even Azure AI Foundry's own managed Mixtral offering?

The full worked answer is Chapter 2. The compressed, interview-ready version:

> "There are three separate reasons, and I want to keep them separate rather than blur them into one
> 'open is better' argument, because they don't all apply to every project — they applied here
> specifically. First, cost at this document volume: per-token API pricing, whether Azure OpenAI or
> Azure AI Foundry's own hosted Mixtral, scales roughly linearly with tokens processed, while
> self-hosted inference cost is dominated by fixed GPU-hour compute — above a real breakeven volume,
> which this platform's document throughput clears, the fixed-cost curve wins. Second, fine-tuning
> control: the client needed Mixtral adapted to their own proprietary regulatory taxonomy, which
> requires direct weight access for LoRA/QLoRA fine-tuning — a managed API's fine-tuning surface, where
> it exists at all, is constrained and vendor-controlled, not something you can point at your own
> confidential taxonomy however you need to. Third, model transparency and audit — this ties directly
> into model-risk-monitoring work I'd also done: a model-risk function that has to formally
> characterize a model's behavior benefits enormously from a fixed, hash-verifiable weight artifact it
> can run its own controlled evaluations against, rather than a black-box API call that could change
> behind the scenes without notice, even a version-pinned one.
>
> And I want to be precise about what I turned down, because it's a stronger option than people assume:
> Mistral's models, including Mixtral, are themselves available through Azure AI Foundry's model
> catalog as a managed API — same weights, hosted by Microsoft. That's a genuinely appealing middle
> path, and I evaluated it. I didn't choose it because reasons two and three — full fine-tuning control
> and weight-level transparency — aren't satisfied by a managed API even when it happens to be serving
> an open-weight model. You get Mixtral as an API call either way; you only get Mixtral as weights you
> control by actually self-hosting it."

**Follow-ups to be ready for:**

- *"What's the actual breakeven volume?"* — Be honest this is a shape-of-the-tradeoff argument, not a
  memorized number: sketch the linear-vs-fixed cost curves and say the real crossover depends on
  current GPU pricing, current API pricing, and actual measured utilization — a calculation worth doing
  explicitly against current numbers, not a fact to recite (Chapter 2).
- *"What if volume turned out lower than expected?"* — Then the cost argument weakens or reverses, and
  it's worth saying so rather than defending self-hosting unconditionally — a badly underutilized
  self-hosted deployment can lose the cost argument entirely (Chapter 2's honest caveat), which is
  exactly why Chapter 4's autoscaling design matters for keeping utilization, and the cost case, honest
  in practice.
- *"Isn't this a lot of extra operational burden for marginal benefit?"* — Only if you undercount the
  benefit: the fine-tuning-control and transparency reasons are not marginal for a regulated client with
  a real proprietary taxonomy and a real model-risk audit requirement — but it's fair to say plainly
  that for a lower-volume, lower-governance-bar workload, a managed API remains the right default, the
  same conclusion courses 1 and 3 reach for their own workloads.

---

### 3. How does Mixtral's architecture compare to GPT-4's or Claude's?

Chapter 3 is the full answer; Chapter 1 supplies the Mixtral-specific technical grounding this answer
leans on. The compressed version:

> "The most important thing to say first is a meta-point: Mixtral's architecture is fully published and
> independently verifiable — decoder-only Transformer, Grouped-Query Attention, Sliding Window
> Attention, and a sparse Mixture-of-Experts design with 8 feed-forward experts per layer and a router
> selecting the top 2 per token, roughly 47 billion total parameters with only about 13 billion active
> per forward pass. Every one of those claims is checkable against Mistral's own technical report. GPT-4's
> and Claude's architectures are not disclosed at that level of detail by OpenAI or Anthropic —
> that's the real, structural difference, not a matter of degree.
>
> On the specific axis people usually ask about — does GPT-4 also use Mixture-of-Experts — I want to be
> precise rather than overclaim: it's widely reported and speculated that it does, but OpenAI has never
> officially confirmed that, so I'd state it as 'reported, not confirmed,' not as settled fact the way
> Mixtral's MoE design is. Claude's architecture is fully undisclosed either way — no public claim about
> MoE or otherwise. Both GPT-4 and Claude are closed-weight, API-only, with no equivalent to Mixtral's
> Apache 2.0 license that lets you actually self-host and fine-tune the weights directly — a closed
> model's 'fine-tuning' is whatever constrained surface the vendor chooses to expose.
>
> And I'd close by being honest about capability: I'm not claiming Mixtral is as capable as the latest
> GPT-4-class or Claude-class model on every benchmark — it's very likely not, at true frontier parity.
> The reason to choose it here wasn't raw intelligence, it was architectural transparency, cost at this
> platform's volume, and fine-tuning control for a narrow, well-scoped classification task — where a
> smaller, well-adapted model closes much of the capability gap anyway. Overselling the open model's raw
> capability would be a weaker answer than being precise about what it actually bought us."

**Follow-ups to be ready for:**

- *"Have you looked at the Claude/GPT-4 comparison in more depth?"* — Point to course 3's dedicated
  chapter on that comparison (chapter 08) rather than re-deriving it — that's the deep dive on the
  closed-model side specifically.
- *"If Mixtral isn't frontier-capable, why not fine-tune a smaller dense model instead of an MoE?"* — A
  fair question: the MoE design still buys more total learned capacity than a dense model with the same
  ~13B active-parameter compute budget (Chapter 1's active-vs-total argument), so for the same inference
  cost, Mixtral likely has more capacity to specialize into the taxonomy during fine-tuning than an
  equivalently-cheap dense model would — that's a defensible reason to prefer it over a dense model at
  the same active-parameter budget, separate from the self-hosting-vs-API argument entirely.

---

### 4. How do you know the model weights you downloaded and deployed are the genuine, untampered Mixtral weights Mistral actually published?

This is the question most unique to this course in the whole curriculum — no other project here takes
custody of model weights at all. Chapter 6 is the full answer:

> "This is a real, non-hypothetical concern once you're self-hosting rather than calling an API — you're
> pulling a large third-party binary artifact into a regulated institution's production infrastructure,
> and nothing about 'it's open-source' guarantees the specific file you downloaded is bit-for-bit what
> Mistral actually published. It could be a corrupted transfer, a compromised mirror, or — the more
> concerning case — a checkpoint that's been subtly modified and redistributed under Mixtral's name,
> designed to behave normally on almost every input except specific trigger cases. For a bank's
> compliance pipeline, a tampered classification model isn't a hypothetical — it's a real model-risk
> exposure if it went undetected to production.
>
> The concrete mitigation is checksumming: before any downloaded checkpoint is ever loaded into the
> serving layer or used as a fine-tuning base, hash it — SHA-256 over the weight files — and compare
> that hash against Mistral's own published reference value. A mismatch is a hard stop, not a warning.
> And that verification isn't a one-time gate — the result is carried durably in the model registry
> entry for every deployed artifact: the weights hash, the publisher's reference hash, when it was
> verified, and where it came from. That registry entry is also where I'd tie in the taxonomy-version
> tag from the staleness design, so one record answers both 'is this genuinely the artifact we think it
> is' and 'is what it was trained to do still current' together.
>
> I'd also point out the asymmetry that makes this worth taking seriously: a compromised open-source
> library dependency usually has a well-understood blast radius — a known CVE, a known vulnerable
> function. A compromised model checkpoint behaves like normal, correct output on the vast majority of
> inputs by design, which is exactly what makes a targeted tampering attack hard to catch through
> ordinary downstream evaluation alone — the fix has to happen at the supply-chain layer, verifying
> before you ever run it, not relying solely on behavioral testing to catch it after the fact."

**Follow-ups to be ready for:**

- *"What if the hash doesn't match — is that always tampering?"* — No, and worth saying so: it's often
  a corrupted or partial download, a mirror serving a different (but legitimate) model revision than
  expected, or a version mismatch against what the deployment config assumed. The response is the same
  either way — hard stop, don't deploy — but the investigation differs: re-download and re-verify first,
  escalate as a security incident only if a clean re-download still mismatches.
- *"Does hash verification catch a compromised checkpoint that Mistral itself published?"* — No — hash
  verification only proves you have exactly what the publisher released; it says nothing about whether
  the publisher's own release process was itself compromised upstream. That's a genuine limit worth
  naming rather than overselling the mitigation, and it's part of why weight-level transparency
  (Chapter 2, Reason 3) is framed as reproducible, independently-evaluable access, not an absolute
  guarantee of correctness.
- *"Is this specific to Mixtral, or would it apply to any open-weight model?"* — Fully general — this
  applies to any open-weight model an organization self-hosts (LLaMA, included — course 11's LLaMA 3
  deployment doesn't cover this angle explicitly, which is exactly why it's this course's distinctive
  contribution to the curriculum rather than a repeat of an existing chapter).

---

### 5. What happens when the client's regulatory taxonomy changes but the fine-tuned model doesn't get retrained?

Chapter 7 is the full worked answer. The compressed version:

> "The naive assumption is that once a fine-tuned model is deployed and passing its evaluation set,
> it's just 'the' model, indefinitely. The gap is that the taxonomy itself isn't static — the client's
> compliance function adds new regulatory categories, deprecates old ones, splits broad categories into
> more specific ones — and none of that touches the model registry or the serving endpoint at all,
> because the taxonomy lives in a completely separate system. The fine-tuned adapter keeps running,
> unaware, and for a document that should now map to a category it's never seen, it confidently returns
> its closest old guess rather than an honest 'I don't recognize this' — nothing about that looks like
> an error. It runs cleanly, produces a classification, and the wrong category flows downstream until a
> compliance reviewer working the actual document catches the mismatch.
>
> The distinction I'd draw precisely, because it's easy to conflate under interview pressure: whether the
> base model version is current, and whether the fine-tuning is current relative to the taxonomy, are
> two completely independent axes. The base model could be the very latest Mixtral release, hash-verified
> and everything, while the fine-tuning is badly stale against a taxonomy that's changed three times
> since the last retrain — or the reverse. Treating 'is the model up to date' as one undifferentiated
> question misses that these have different owners, different triggers, and different failure shapes.
>
> The design I'd propose: a taxonomy-version tag carried on every training example and every model
> registry entry, alongside the weights-hash provenance record from the supply-chain question — so a
> registry lookup tells you immediately which taxonomy version a deployed adapter was trained against.
> And a scheduled drift-detection job that re-runs the current production adapter against a held-out
> validation set kept current against the live taxonomy, comparing accuracy per category against the
> adapter's own training-time evaluation numbers — so a material drop is caught by a scheduled job, not
> by a compliance reviewer stumbling onto a misclassified filing."

**Follow-ups to be ready for:**

- *"Wouldn't retraining constantly, every time the taxonomy changes even slightly, be wasteful?"* — Fair
  — the design should batch non-urgent taxonomy edits and trigger retraining on a cadence (or once
  enough changes accumulate) rather than a full retrain on every single edit; what shouldn't wait is
  detection — the drift-detection job should flag the mismatch immediately even if the actual retrain is
  scheduled for a batched cadence, so the gap is visible and tracked, not silently accruing.
- *"How is this different from course 1's knowledge-freshness problem or course 11's protocol-amendment
  problem?"* — Same underlying shape (a dependency changes, nothing notices automatically), different
  domain and a genuinely two-dimensional version of it here — course 1's problem is single-axis (is the
  retrieval index current), this one is explicitly two independent axes (base model vs. fine-tuning
  currency) that can drift in either direction independently, which is the specific nuance worth
  volunteering rather than just pattern-matching to the familiar single-axis staleness answer.

---

### 6. Walk me through what happens, end to end, when a regulatory filing enters this pipeline.

Document lands in Azure Blob Storage (ingestion) → the orchestration service picks it up → it's chunked
respecting section boundaries for long filings → the hybrid retrieval layer (Azure AI Search) surfaces
related policy context → a classification/extraction call goes to the self-hosted Mixtral endpoint
(Azure ML managed online endpoint or AKS+vLLM, Chapter 4), running the fine-tuned LoRA adapter for the
client's taxonomy (Chapter 5) → structured output (taxonomy code, extracted fields) is schema-validated
→ on success it's routed to the downstream compliance workflow for human review; on a schema failure or
low-confidence result it's routed to mandatory manual review instead (Chapter 8's error-handling table)
→ every stage is logged to Azure Monitor, and the model registry entry used for that call (weights hash,
adapter version, taxonomy version) is attached to the audit record. See `00-README.md`'s architecture
diagram for the full picture.

---

### 7. How would you reduce hallucination/misclassification risk in this pipeline specifically?

Layered, similar in spirit to course 1's hallucination-reduction stack but adapted to a
classification/extraction task rather than open-ended chat: (1) grounding classification decisions in
retrieved related-policy context via the hybrid retrieval layer, not just the raw filing text alone
(Chapter 4); (2) schema-constrained structured output so a malformed or incomplete response is
detectable mechanically, not just by eyeballing prose (Chapter 8); (3) a domain-specific evaluation set
scored per-category and per-field, not aggregate-only, so narrow regressions (Chapter 8's quantization
bug) don't hide behind a healthy overall number; (4) mandatory human review as the default for the
downstream compliance workflow, never generate-and-publish — the same human-in-the-loop principle
course 2 and course 11 both build their entire architecture around; (5) confidence-aware routing — a
low-confidence classification routes to manual review rather than being silently accepted.

---

### 8. How would you evaluate this platform before and after a fine-tuning run or a serving-configuration change (like a quantization change)?

Beyond the domain evaluation set itself (Chapter 5) — per-category classification accuracy/F1 and
per-field extraction accuracy against a held-out, expert-labeled set — the discipline worth stating
explicitly is that **any** change to what's actually running deserves the same evaluation rigor, not
just adapter changes: a quantization configuration change (Chapter 4, Chapter 8's Bug 1) is a change to
how the same weights compute, and it needs the same per-category regression check as a new fine-tuned
adapter, because an aggregate-only check would have shipped that regression to production. This is the
direct analog of course 4's Ragas-based evaluation discipline, applied to a classification/extraction
task shape instead of RAG faithfulness/relevance scoring.

---

### 9. How would you scale this platform to handle a large, sudden spike in filing volume — say, around a major new regulation's effective date?

This is where the self-hosted-vs-managed-API distinction matters most concretely (Chapter 8): unlike a
managed API backed by a provider's enormous shared compute fleet, this platform's Mixtral endpoint has a
hard capacity ceiling set by whatever GPU instances are actually provisioned. The mitigations, layered:
(1) autoscaling triggered on GPU utilization and queue depth, not a generic CPU metric that would miss
the actual bottleneck (Chapter 4, Chapter 8's Bug 3); (2) a bounded request queue with explicit
backpressure so the system fails predictably under genuine overload rather than degrading silently or
crashing; (3) capacity planning that explicitly budgets for known burst patterns — regulatory deadlines
are knowable in advance, similar to course 11's protocol-amendment-deadline bursts — rather than sizing
only for average daily volume; (4) as a last-resort overflow valve worth naming honestly, a documented
fallback path to a managed API (even at a cost premium) for genuinely unplanned demand that exceeds
provisioned self-hosted capacity, rather than pretending the self-hosted ceiling never gets tested.

---

### 10. Why LoRA/QLoRA instead of full fine-tuning for adapting Mixtral to this taxonomy?

Course 11's chapter 02 covers the mechanics in depth — this course applies rather than re-derives it
(Chapter 5). The short version: full fine-tuning of a ~47B-parameter model means updating and storing
gradients/optimizer state for every parameter, which is expensive and produces a full new model copy
per specialized behavior. LoRA freezes the base weights and learns a small low-rank update on top,
often under 1% of the parameters of a full update, making it practical to train (and store) a per-
category or per-taxonomy-version adapter cheaply, and to swap adapters on top of one shared frozen base
rather than hosting multiple full model copies. QLoRA additionally quantizes the frozen base during
training, which matters extra for Mixtral specifically because even though only ~13B parameters are
active per forward pass (Chapter 1), the full ~47B still has to be loaded into memory during training
since every expert must be present — QLoRA's memory saving there is real, not marginal.

---

### 11. What's the difference between Azure ML managed online endpoints and AKS with vLLM/TGI — and which would you pick?

Chapter 4 has the full tradeoff table. The compressed version: Azure ML managed online endpoints trade
some control for a much lower operational burden — Azure handles compute provisioning, built-in
autoscaling, and traffic splitting, and you supply a container and a compute SKU. AKS with vLLM or TGI
gives full control over batching, KV-cache management, and multi-adapter scheduling, at the cost of
owning the Kubernetes cluster and the serving framework's operations yourself. For this platform's
actual profile — sustained, high document-processing volume but not latency-critical interactive chat —
a defensible starting position is Azure ML managed online endpoints for the initial rollout, explicitly
evaluated against a migration to AKS + vLLM once real measured production volume and throughput
requirements (not estimates) justify the added operational complexity.

---

### 12. What's quantization actually costing you, and how would you decide how aggressively to quantize?

Quantization reduces weight precision (e.g., 16-bit to 8-bit or 4-bit), cutting memory footprint roughly
proportionally and often improving throughput since less data moves through memory bandwidth — directly
useful for fitting Mixtral on a smaller/cheaper GPU tier or freeing KV-cache headroom for more
concurrency (Chapter 4). The cost is a real, non-uniform quality degradation — it doesn't hit every task
type evenly, which is exactly what Chapter 8's Bug 1 illustrates: an 8-bit configuration that looked fine
in aggregate quietly hurt accuracy on one specific, less-common taxonomy category. The only responsible
way to choose a quantization level is measuring quality against the actual per-category domain
evaluation set (Chapter 5) at each candidate level, not trusting a published general-benchmark quality
delta — quantization level is a tunable point on a cost/latency/quality curve, not a binary switch.

---

### 13. This platform serves a bank's compliance/legal function — what does "production-grade" mean here beyond correct classifications?

Layered, mirroring course 1's answer to the analogous question but adapted to this platform's specific
architecture: (1) the same edge/network protections as every Capco course — VNet, Private Endpoints, no
backend service reachable from the public internet; (2) Azure AD/RBAC gating not just document upload
but inference access and model-registry write access separately, since triggering a retrain is a much
higher-privilege action than submitting a document for classification (Chapter 6); (3) full artifact
provenance — every deployed model traceable to a hash-verified weights source and an explicit taxonomy
version (Chapter 6, Chapter 7); (4) mandatory human review before anything reaches a real compliance
decision, never generate-and-publish; (5) an honest capacity story — a self-hosted GPU serving layer has
a real ceiling, and production-grade means planning for it explicitly rather than assuming elastic
capacity that doesn't exist here (Chapter 8).

---

### 14. What would you change if you rebuilt this today?

A few concrete things: First, I'd build the taxonomy-version-tagged drift-detection job (Chapter 7) in
from day one rather than treating it as a later addition — the whole point is catching drift before a
human catches it manually, and that only works if it exists before the first taxonomy change happens,
not after. Second, I'd extend artifact provenance to cover the tokenizer explicitly from the start
(Chapter 8's named hardening gap) rather than discovering the gap through an actual tokenizer-mismatch
incident — it's a small, obvious extension of infrastructure (hash verification) that already exists for
the base model. Third, I'd load-test the autoscaling policy itself as a first-class test, not just the
serving stack's raw throughput — Chapter 8's CPU-vs-GPU-utilization autoscaling bug is exactly the kind
of thing that only shows up under a genuine sustained-load test, not a functional-correctness test.
Fourth, I'd build per-category evaluation scoring as a hard gate on *any* serving-configuration change
(quantization included), not just adapter changes — Bug 1 in Chapter 8 exists specifically because that
distinction wasn't drawn early enough.

---

### 15. If you'd chosen Azure AI Foundry's managed Mixtral instead of self-hosting, what would you have actually lost?

A precise, specific answer, not a vague "less control": you'd lose direct LoRA/QLoRA fine-tuning access
to the client's proprietary taxonomy — you'd be limited to whatever fine-tuning surface Azure AI
Foundry exposes for that model, if any, on Microsoft's terms rather than your own infrastructure. You'd
lose weight-level provenance and transparency — the model-risk function could evaluate the API's
external behavior, but couldn't hash-verify or independently inspect the actual served weights, and
couldn't be certain the served model hadn't changed underneath an unpinned or even a pinned-but-
externally-controlled endpoint. And you'd lose the cost profile at volume — you'd be back on a
per-token pricing curve rather than a fixed-compute one, which matters directly at this platform's
processing volume. What you'd gain: essentially zero infrastructure operations burden, no GPU sizing or
autoscaling decisions (Chapter 4), no serving-stack bugs to debug (Chapter 8), and no supply-chain
verification responsibility (Chapter 6) — those become Microsoft's problem entirely. That's a completely
legitimate tradeoff for a lower-volume, lower-governance-bar project; it just wasn't the right one here.

---

### 16. How is this project different from course 11's LLaMA 3 deployment — aren't they both "self-hosted open-weight LLM" stories?

Structurally similar at a high level — both are open-weight, self-hosted, LoRA-fine-tuned deployments —
but genuinely different in the details worth naming precisely rather than treating as interchangeable:
course 11's LLaMA 3 deployment is on **AWS Sagemaker**, for **generation** tasks (drafting ICF/PLPS/SOC
content in a target register), with data-residency/IP-protection as the primary self-hosting driver
(two competing pharma clients' unpublished protocols). This platform is on **Azure ML/AKS**, for
**classification/extraction** tasks against a proprietary taxonomy, with cost-at-volume,
fine-tuning-control, and weight-level-audit-transparency as the three drivers — data residency wasn't
the leading argument here the way it was for course 11, since the Capco banking client base's Azure
tenant boundary already handles residency for every other course in the curriculum via Azure OpenAI's
in-tenant deployment model. This project's genuinely novel contribution — model supply-chain/provenance
verification (Chapter 6) — isn't something course 11's chapters cover, which is worth naming directly if
asked to distinguish the two.

---

### 17. What's a bug you found and fixed that was specific to self-hosted LLM serving, not generic backend work?

Pick one of the four from Chapter 8 — the autoscaling-on-CPU-instead-of-GPU bug is a strong, concrete
choice because it's a mistake that's easy to make by inheriting a generic policy template and hard to
notice until real load hits: an early AKS-based serving deployment inherited an autoscaling policy from
an unrelated CPU-bound service elsewhere in the client's Azure estate, scaling replica count on CPU
utilization. Under real inference load, GPU utilization was consistently saturated while CPU stayed low
— the orchestration overhead is genuinely CPU-light relative to GPU-bound inference — so the autoscaler
never triggered a scale-out even as request queues grew and latency degraded, because it was watching a
metric that wasn't the actual bottleneck. What would have caught it earlier: load-testing the
autoscaling policy itself, specifically verifying a sustained synthetic load actually triggers a
scale-out event, not just confirming the serving stack works correctly at a fixed replica count.

---

### 18. How would you explain the router mechanism in Mixtral's MoE layers to someone unfamiliar with the architecture?

At each Transformer layer, instead of one shared feed-forward block processing every token, there are 8
separate feed-forward "expert" blocks. A small router network looks at each token's hidden state and
produces a relevance score for each of the 8 experts, then only the top 2 scoring experts actually
process that token — the other 6 sit idle for it, at that layer. The two selected experts' outputs are
combined, weighted by the router's own confidence in each. This routing decision happens independently
at every layer, so a token isn't permanently "assigned" to one expert across the whole model — it's a
fresh, local decision each layer. See Chapter 1 for the full walkthrough, and
`notebooks/01_moe_expert_routing_demo.ipynb` for a from-scratch numpy implementation of exactly this
mechanism.

---

### 19. Is it accurate to say Mixtral "has 47 billion parameters"? What does that actually mean given the MoE design?

Precisely: Mixtral 8x7B has roughly 47 billion **total** parameters — that's everything: all 8 experts
at every layer, the attention layers, the router, the embeddings. But because the sparse MoE routing
only activates the top 2 of 8 experts per token at each layer, only roughly 13 billion parameters are
actually **active** — actually doing compute — on any single token's forward pass. The "8x7B" name is a
useful mnemonic but not a literal parameter count (a naive 8×7B would be 56B; the real total is lower
because attention layers, embeddings, and the router are shared across experts rather than duplicated).
The 47B/13B split is the entire efficiency argument for the MoE design: access to more total learned
capacity than a dense ~13B model, at the inference compute cost of a dense ~13B model, not a dense ~47B
one. See Chapter 1 and `notebooks/02_active_vs_total_parameter_cost_demo.ipynb`, which computes the
FLOPs implication of this split numerically rather than just asserting the efficiency claim.

---

### 20. What would concern you most about this architecture if you were the client's model-risk reviewer, not the engineer who built it?

A strong answer names something genuinely uncomfortable rather than a softball: the router's
specialization pattern isn't cleanly interpretable — published analysis of Mixtral's routing behavior
shows it doesn't obviously specialize by topic the way an intuitive story might suggest (Chapter 1), so
"which expert handles what" isn't something you can explain cleanly to an auditor even with full weight
access. Weight-level transparency (Chapter 2, Reason 3) buys reproducibility and independent
evaluability — you can run the same fixed artifact repeatedly and know it hasn't silently changed — but
it does not automatically buy mechanistic interpretability the way a simpler classical model's
Shapash/LIME explanation does (course 4, chapter 3). A rigorous model-risk review has to be honest that
"we can inspect the weights" and "we can fully explain every output" are different claims, and only the
first one is actually true here.

---

### Quick follow-up probes to be ready for

- "What's Grouped-Query Attention buying you, concretely?" → A smaller KV cache and faster inference by
  sharing key/value projections across groups of query heads, at a small expressiveness cost relative
  to full multi-head attention — Chapter 1.
- "What's Sliding Window Attention trading away?" → Full quadratic attention becomes fixed-window local
  attention at each layer; the model regains an indirect longer effective span through the stack of
  layers, but any single layer only sees a bounded local window directly — Chapter 1.
- "Is Mixtral's MoE the same idea as GPT-4's rumored MoE?" → Structurally the same general family of
  idea (sparse expert routing), but GPT-4's specific design (if any) has never been officially confirmed
  by OpenAI — don't claim more precision than exists publicly — Chapter 3.
- "What's the difference between the base-model-weights hash and the taxonomy-version tag on a registry
  entry?" → The hash answers "is this genuinely the artifact we think it is" (supply-chain provenance,
  Chapter 6); the taxonomy-version tag answers "is what it was trained to do still current" (fine-tuning
  currency, Chapter 7) — genuinely independent axes carried on the same record.
- "Would you ever run both a self-hosted Mixtral endpoint and a managed API side by side?" → Yes, and
  worth naming as a realistic design rather than an either/or: the self-hosted endpoint for the core
  high-volume, taxonomy-fine-tuned classification/extraction work, with a managed API kept as a
  documented overflow/fallback path for genuine capacity spikes (Chapter 8) or for exploratory tasks not
  yet worth building fine-tuning data for — the same "both, for different workloads" framing course 11
  reaches for LLaMA 3 vs. AWS Bedrock.
