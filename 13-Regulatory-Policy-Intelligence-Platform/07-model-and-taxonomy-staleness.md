# 07 — Model and Taxonomy Staleness

## Why this chapter exists

This is this course's version of the staleness/versioning chapter every course in this curriculum
carries in some form — course 1's knowledge-freshness chapter, course 4's baseline-staleness chapter,
course 11's protocol-amendment-versioning chapter. Each is the same underlying pattern applied to a
different domain: something the system depends on changes, nothing automatically notices, and a
downstream user eventually catches the drift the hard way. Here, the twist is that there are **two
independent things that can go stale**, not one — the base model version, and the fine-tuned adapter's
currency relative to the client's regulatory taxonomy — and conflating them is the specific mistake this
chapter is built to prevent.

## Part 1 — The naive assumption

The naive mental model, easy to hold implicitly even by an engineering team that would reject it if
stated out loud: **the deployed model is "the" model, indefinitely.** Once Mixtral is fine-tuned on the
client's taxonomy, evaluated, and promoted to production (Chapter 5), it's tempting to treat that as a
finished, durable state — the model registry has an entry, the endpoint is serving it, the domain
evaluation set passed. Nothing in that picture forces anyone to revisit whether the model is still
correct *relative to a taxonomy that keeps changing underneath it*.

## Part 2 — The gap

**A taxonomy change without a corresponding retrain silently degrades classification accuracy on the
new/changed categories, and nobody notices until a downstream compliance reviewer catches a
misclassification.** Concretely: the client's compliance function adds a new regulatory category (a
newly enacted regulation needs its own taxonomy code), deprecates an old one (superseded guidance), or
splits an existing broad category into two more specific ones. None of that requires touching the
deployed model at all — the taxonomy lives in the client's policy/compliance systems, entirely separate
from the model registry and the serving endpoint. The fine-tuned adapter keeps running, unaware, and
keeps producing its best guess at a classification using categories it was actually trained on — which,
for a document that should now map to a brand-new category the adapter has never seen, means it
confidently emits the *closest old category it knows*, not an honest "I don't recognize this," and
nothing about that failure looks like an error. The pipeline runs, a classification is produced, a
confidence score (if surfaced at all) may look unremarkable, and the output flows downstream exactly as
if it were correct — until a compliance reviewer, working the actual document, notices the assigned
category doesn't match the new regulation it clearly falls under.

This is structurally the same shape of bug as course 1's cache-serving-a-stale-answer problem and
course 11's protocol-amendment-without-a-corresponding-SOC-update problem: **the system has no signal
that its own grounding has gone stale, so it keeps confidently operating on outdated information.**

## Part 3 — Two concerns that must be distinguished: base-model currency vs. fine-tuning currency

This is the distinction this chapter exists to draw precisely, the same way course 1's chapter 6 draws
the line between conversation-turn-count and knowledge-freshness, and course 5's chapter 5 draws the
line between approval-lifecycle and content-revision. Here, the two axes are:

| | **Base-model version currency** | **Fine-tuning-vs-taxonomy currency** |
|---|---|---|
| What it tracks | Whether the deployed Mixtral base weights are the version the team intends to run (Mistral could ship a new Mixtral release; course 3's chapter 06 covers the general version-pinning risk this maps onto) | Whether the fine-tuned adapter's learned category boundaries reflect the client's *current* taxonomy |
| Who/what changes it | Mistral AI, by releasing a new model version | The client's compliance/legal function, by editing their taxonomy |
| Detection today | Model registry's pinned weights hash (Chapter 6) — a version mismatch is immediately checkable | Nothing automatic — the adapter has no way to know the taxonomy it was trained against is no longer the live one |
| Failure if unaddressed | Running an outdated base model, missing whatever general-capability improvements a newer Mixtral release carries — a capability/quality gap, not a correctness gap on any specific category | Confidently misclassifying documents against categories that no longer exist, or failing to recognize categories that didn't exist at training time — a correctness gap, potentially invisible until a human catches it |

**These two can drift completely independently of each other**, and it's worth stating plainly: the
base model could be perfectly current (latest Mixtral release, hash-verified against Chapter 6's
process) while the fine-tuning is badly stale relative to a taxonomy that changed three times since the
last retrain — or the reverse, an aging base model paired with a fine-tuned adapter retrained just
yesterday against the current taxonomy. Treating "is the model up to date" as one undifferentiated
question, rather than two separate ones with separate triggers and separate owners, is exactly the kind
of imprecision that makes an otherwise-solid staleness answer sound shallow under interview follow-up.

## Part 4 — A proposed design: taxonomy-version tagging and drift detection

Explicitly a proposal, in the same spirit as every other course's staleness-chapter design — not a
claim that this exists as shipped infrastructure.

**1. A taxonomy-version tag propagated through training data and model registry entries.** Every
labeled training/evaluation example (Chapter 5) is tagged with the taxonomy version it was labeled
against. Every model registry entry — building directly on Chapter 6's provenance record — carries that
same taxonomy-version tag alongside the base-model-weights hash:

```json
{
  "artifact_id": "mixtral-8x7b-regtaxonomy-adapter-v14",
  "base_model_weights_sha256": "a3f9...c21e",
  "taxonomy_version": "reg-taxonomy-v7.2",
  "trained_at": "2026-05-02T00:00:00Z",
  "domain_eval_result": {"overall_f1": 0.91, "worst_category_f1": 0.78}
}
```

This single record now answers both staleness questions at once: is the base model the intended,
hash-verified version (Chapter 6), and exactly which taxonomy version was this specific adapter trained
against — so a registry lookup, not institutional memory, tells you immediately whether a given
deployed artifact's fine-tuning is current relative to whatever the taxonomy's live version is right
now.

**2. A scheduled drift-detection job comparing live classification distributions against a held-out
validation set tagged to the current taxonomy version.** Rather than waiting for a human reviewer to
catch a misclassification, a periodic job:

- Pulls a sample of recent production classifications and compares the distribution of assigned
  taxonomy categories against expected/historical distributions, flagging categories whose volume has
  shifted unexpectedly (a proxy signal — a category that should be receiving documents but suddenly
  isn't, or the reverse, is worth investigating even before confirming a root cause).
- Re-runs the current production adapter against a held-out validation set that is itself kept current
  against the live taxonomy version (maintained by the same compliance/legal review process that
  maintains the taxonomy), and compares accuracy per-category against the adapter's own
  `domain_eval_result` at training time — a material drop signals the adapter is stale relative to
  what the taxonomy now expects, even before any specific misclassification has been caught downstream.
- Surfaces a simple, explicit signal: **taxonomy version mismatch** (the live taxonomy version has
  advanced past the deployed adapter's `taxonomy_version` tag) as a standing, checkable fact, not
  something inferred indirectly from accuracy metrics alone.

Notebook `04_taxonomy_drift_detection_demo.ipynb` implements this comparison directly: a synthetic
"training-time" taxonomy distribution against a "current" one, flagging categories that have drifted
beyond a threshold.

## Part 5 — Tying forward to Chapter 6's provenance tracking

The registry-entry design above isn't a new proposal invented in isolation — it's the direct extension
of Chapter 6's provenance record. Chapter 6 established that every deployed artifact should carry a
hash-verified base-model provenance record so its *origin* is never in question. This chapter adds the
second half: that same registry entry should carry the taxonomy version the fine-tuning was performed
against, so its *currency* relative to a changing business requirement is never in question either.
Provenance answers "is this genuinely the artifact we think it is"; the taxonomy-version tag answers "is
what it was trained to do still what we need it to do" — both live on the same registry entry precisely
because a model-risk reviewer, or an automated drift-detection job, needs both answers together to fully
characterize a deployed model, not one without the other.

## Tying It Back

If asked "what happens when the taxonomy changes but the model doesn't get retrained": the honest answer
starts with the naive assumption (the deployed model is treated as permanently correct), names the real
gap precisely (a taxonomy change is invisible to the model and produces confident, not obviously wrong,
misclassifications), draws the base-model-currency-vs-fine-tuning-currency distinction explicitly rather
than treating "staleness" as one undifferentiated concern, and proposes a concrete, checkable design —
taxonomy-version tags carried on every training example and registry entry, plus a scheduled drift-
detection job comparing live classification distributions and accuracy against a taxonomy-version-
tagged validation set — that catches the drift before a compliance reviewer has to catch it manually.
