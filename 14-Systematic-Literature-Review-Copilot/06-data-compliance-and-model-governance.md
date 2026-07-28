# 06 — Data Compliance and Model Governance

## Data handling: licensed literature and patient-level data in source studies

Two distinct categories of sensitive content flow through this pipeline, and they call for different
handling even though both land in the same S3-backed corpus.

**Licensed literature-database content.** Much of an SLR's source corpus comes from databases like
PubMed (largely open) but frequently also from **client-licensed sources** — subscription
literature databases, publisher full-text access, or proprietary internal evidence repositories a
pharma client has separately licensed. That licensing relationship carries real contractual
obligations: redistribution restrictions, retention limits, and usage-scope limits (e.g., licensed
for this client's internal review use, not for training a model that might later serve a different
client). The practical implication for this pipeline's design: **per-client S3 bucket/prefix
isolation with per-client IAM scoping**, the same pattern course 11's chapter 00 establishes for
protocol documents between two competing pharma clients — a licensed-content access agreement made
with one client's literature vendor has no bearing on, and must never be readable by, another
client's pipeline, and a fine-tuning dataset built from one client's licensed corpus must not leak
into another client's model artifact.

**Patient-level data appearing in source studies.** Published clinical literature occasionally
includes case reports, small case series, or supplementary data with patient-level detail — rare
relative to the bulk of aggregate-level published findings, but not zero, and worth explicit handling
rather than an assumption that "it's published, so it's already de-identified and safe." The
governing principle, consistent with course 11's patient-safety and human-review framing rather than
re-explained here: any patient-level detail encountered flows through the same conservative,
human-reviewed handling this curriculum applies to patient-facing content generally — flagged for
reviewer attention rather than processed and forgotten, and never used as fine-tuning training
material without the same scrutiny any patient-level data would receive under the client's own data-
handling policies. This is a narrower concern than course 11's ICF/PLPS generation (which produces
patient-facing content directly), but the underlying discipline — treat anything patient-level as
requiring deliberate, reviewed handling, never silent default processing — is the same one.

## The self-hosted-model-specific concern: model supply-chain and provenance verification

This is the angle that's specific to self-hosting an open-weight model, distinct from any data-
handling concern above, and it deserves to be treated as a first-class governance question rather than
an afterthought: **how do you actually know the DeepSeek weights your pipeline downloaded and deployed
are genuine** — the exact weights DeepSeek published, unmodified — **and not a compromised, tampered,
or subtly altered checkpoint** obtained from an untrusted mirror, a compromised download path, or
(harder to detect) a checkpoint someone has deliberately altered to introduce a hidden behavior?

This is not a hypothetical concern invented for this chapter. Any organization that self-hosts
open-weight model checkpoints downloaded from the internet faces a genuine software-supply-chain
question, structurally analogous to verifying a downloaded software package or container image
against a publisher's signature before running it in production — except a model checkpoint is a
large binary blob of numeric weights, not human-readable code, which makes a casual "eyeball it and
see if it looks right" review meaningless. A tampered checkpoint could, in principle, behave
identically to the genuine model on the vast majority of inputs while producing subtly different
behavior on specific triggers — exactly the kind of integrity failure that's hard to catch through
functional testing alone and easy to catch through **checksum/hash verification against the
publisher's published reference values**.

The concrete practice this implies, worth stating plainly as a governance requirement rather than a
nice-to-have: **every model artifact deployed to a SageMaker endpoint or batch job is verified against
a cryptographic hash (e.g., SHA-256) published by DeepSeek for that exact release**, before it's ever
promoted into a pipeline whose output feeds regulatory-submission-supporting evidence. A mismatch
between the computed hash of the downloaded artifact and the publisher's published reference value is
a **hard stop** — the artifact is quarantined, not deployed, and the download investigated, never
silently accepted with a logged warning. This is the same category of hard-block discipline course
11's chapter 07 applies to content-integrity failures (grounding checks, schema validation): an
unverified model artifact is strictly worse than no model artifact at all, the same way an unversioned
generated document was strictly worse than no document at all in that chapter's error-handling table.

Provenance tracking extends past the initial download: the verified hash, the download source, the
verification timestamp, and the resulting model-version identifier (Chapter 7) are recorded together
as a permanent artifact record — so months later, if a question arises about a screening decision
made under a specific model version, the chain from "this model version" back to "this exact,
verified checkpoint, downloaded from this source, on this date, matching DeepSeek's published hash"
is fully reconstructable. That reconstructability is what actually makes Chapter 2's Reason 3
(regulatory auditability) operationally real rather than a claim — a fixed, versioned model artifact
is only as trustworthy as the provenance chain proving it's genuinely the artifact it claims to be.

## Notebook: making this concrete

`notebooks/05_model_provenance_verification_demo.ipynb` implements exactly this checksum-verification
pattern on synthetic data: hashing a stand-in "model weights" file, comparing it against a "published"
reference hash, demonstrating both a clean match (deploy proceeds) and a tampered/mismatched case
(deployment blocked, discrepancy surfaced) — the same shape as course 13's equivalent provenance
notebook for its own self-hosted open-weight model, adapted here to DeepSeek. Running it side by side
with this chapter makes the abstract "verify before you deploy" governance statement into something
you can point to and say precisely what it checks and precisely what it catches.

## Tying It Back

Data compliance here has two layers that shouldn't be conflated: ordinary (if pharma-specific)
licensed-content and patient-level-data handling, governed by the same per-client isolation and
human-review discipline course 11 establishes, and a **self-hosted-model-specific** layer —
supply-chain provenance verification — that has no analog in a managed-API architecture at all,
because you never download and deploy someone else's API. Verifying every DeepSeek checkpoint against
a publisher-issued hash before it's ever promoted into a pipeline feeding regulatory-submission-
supporting evidence isn't optional hardening; it's the specific mechanism that makes "we know exactly
what model produced this decision" (Chapter 2, Reason 3) a claim backed by evidence rather than
assumption — and it's the piece Chapter 7's model-version tracking depends on being true in the first
place.
