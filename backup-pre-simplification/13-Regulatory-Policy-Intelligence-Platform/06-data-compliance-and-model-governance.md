# 06 — Data Compliance and Model Governance

## Why this chapter exists

Every Capco course in this curriculum has a data-compliance chapter or section — this platform needs
the same baseline (data residency, access control) covered briefly here rather than re-derived, plus
one concern that is genuinely new to this course and doesn't appear anywhere else in the curriculum:
**how do you know the model weights you downloaded and deployed are actually genuine, untampered
Mistral-published weights, and not a compromised or subtly modified checkpoint?** That question only
exists because this project self-hosts open weights at all — none of the other Capco courses, which
call a managed API, have a supply-chain question of this shape, because they never take custody of
model weights in the first place.

## The standard part, briefly: data residency and access control

This platform's document corpus — regulatory filings and internal policy documents — lives in **Azure
Blob Storage** inside the same VNet/Private Endpoint boundary as every other Capco course in this
curriculum, reachable only from inside that boundary, never over the public internet. Authentication
and authorization follow the same pattern course 5 establishes in depth
(`04-authentication-rbac-with-msal-and-oauth.md`): Azure AD-issued tokens via MSAL, RBAC roles
determining what an authenticated identity can actually do (upload a document, trigger a
classification run, approve a classification result, retrain an adapter), rather than a shared secret
or open access. That chapter goes deep on the mechanics; this course reuses the pattern rather than
re-explaining it. The one platform-specific addition: RBAC roles here also gate access to the
**self-hosted model endpoint itself** (Chapter 4's diagram shows this explicitly) and to the **model
registry** (below) — inference access and fine-tuning/registry-write access are separate, narrower
roles, since being allowed to submit a document for classification is a materially lower-privilege
action than being allowed to trigger a retrain or promote a new adapter to production.

## The concern unique to self-hosting open weights: model supply-chain and provenance verification

**The problem, stated plainly.** When you download Mixtral's published weights — from Mistral's own
release channels, from a model hub, from wherever your organization's approved distribution path is —
you're pulling a large binary artifact from a third party into a regulated institution's production
infrastructure. Nothing about "the weights are open-source" guarantees the specific file you downloaded
is bit-for-bit what Mistral actually published. A compromised mirror, a corrupted transfer, a
man-in-the-middle substitution, or — the more insidious case — a checkpoint that's been subtly modified
(a backdoor fine-tune designed to behave normally except on specific trigger inputs, for instance) and
redistributed under Mixtral's name, are all real categories of risk once you're in the business of
downloading and running third-party model weights rather than calling an API where the vendor controls
the served artifact end to end. For a bank's compliance/legal document pipeline specifically, a
tampered classification model is not a hypothetical inconvenience — it's a model-risk-governance
failure with real regulatory exposure if it ever went to production undetected.

**Why this is a genuinely novel point for this curriculum, not a restatement of an existing concern.**
Every other course's data-compliance chapter (course 2's AML compliance content, course 3's vendor-risk
chapter for a new API vendor) is about controlling where *data* flows and who can access it. This is a
different axis entirely: it's about verifying the integrity of the *model artifact itself* before it
ever processes any data at all. Course 11's LLaMA 3 chapter names self-hosting's operational
responsibilities in general terms but doesn't cover artifact-integrity verification specifically — this
is the one place in the curriculum that does, and it's worth naming as such if asked "what's different
about governance for this project versus the other GenAI projects in your background."

**Checksumming and hash verification against the publisher's published model card.** The concrete,
practical mitigation: Mistral publishes cryptographic hashes (or a mechanism to derive them, such as a
model hub's own integrity metadata) alongside its released model artifacts. Before any downloaded
checkpoint is loaded into the serving layer (Chapter 4) or used as the base for fine-tuning (Chapter 5),
it should be hashed (e.g., SHA-256 over the weight files) and that hash compared against the
publisher's published reference value. A mismatch is a hard stop — the artifact is not deployed,
full stop, until the discrepancy is understood. Notebook `05_model_provenance_verification_demo.ipynb`
implements exactly this: hashing a synthetic "model weights" file and comparing it against a "published"
reference hash, demonstrating both a clean match and a tampered/mismatched case being caught.

**Provenance tracking in the model registry.** Hash verification at download time is necessary but not
sufficient on its own — the model registry entry for every deployed artifact (base model or fine-tuned
adapter) should carry a durable provenance record, not just a name and a version number:

```json
{
  "artifact_id": "mixtral-8x7b-base-v1",
  "base_model_weights_sha256": "a3f9...c21e",
  "publisher_reference_hash": "a3f9...c21e",
  "hash_verified_at": "2026-02-11T08:03:00Z",
  "source": "Mistral AI official release channel",
  "downloaded_by": "platform-mlops-service-principal",
  "verification_status": "MATCH"
}
```

For a fine-tuned adapter, this same record extends to include exactly which verified base-model hash it
was trained on top of, and — tying forward to Chapter 7 — exactly which taxonomy version its training
data reflected. That combination (base-model-weights hash + taxonomy version) is the pairing Chapter 7
proposes carrying on every registry entry, because the two can drift independently and a registry entry
that only tracks one of them can't answer "is this exact artifact still correct" on its own.

**Why this is a real, non-hypothetical concern for a regulated institution self-hosting open-weight
models, not a theoretical exercise.** Financial institutions already have mature supply-chain-security
practice for software dependencies (vetted package registries, SBOM/dependency scanning, signed
artifacts) — treating a downloaded model checkpoint with less rigor than a downloaded software
dependency would be an inconsistency a model-risk or infosec review would reasonably flag. The stakes
are also asymmetric in a way worth stating directly: a compromised open-source *library* dependency
typically has a well-understood blast radius (a known CVE, a known vulnerable function); a compromised
*model checkpoint* behaves like ordinary correct output on the vast majority of inputs by design — that's
what makes a targeted, trigger-based tampering attack against a model artifact genuinely hard to detect
through normal evaluation alone, and exactly why the fix has to happen at the supply-chain layer
(verify before you ever run it) rather than relying solely on downstream behavioral evaluation to catch
it after the fact.

## Tying It Back

The standard data-compliance content here is genuinely standard — same VNet/Private Endpoint boundary,
same Azure AD/RBAC pattern course 5 establishes, applied to this platform's document corpus and model
endpoint. What's distinctly this course's own contribution to the curriculum is the supply-chain angle:
self-hosting open weights means taking on artifact-integrity responsibility a managed-API project never
has to think about, and the concrete answer — hash-verify every downloaded checkpoint against the
publisher's reference before deployment, and carry that verification record durably in the model
registry alongside the taxonomy-version tag Chapter 7 depends on — is a real, defensible answer to a
question no other course in this curriculum needs to ask.
