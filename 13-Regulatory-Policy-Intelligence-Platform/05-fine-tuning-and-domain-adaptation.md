# 05 — Fine-Tuning and Domain Adaptation

## Why this chapter exists

Chapter 2's second reason for self-hosting was fine-tuning control — the client needs Mixtral adapted
to their own proprietary regulatory taxonomy and internal policy classification scheme, which requires
direct access to model weights. This chapter is where that control actually gets used: how you'd apply
LoRA/QLoRA fine-tuning to Mixtral for this platform's classification and extraction tasks, how you'd
build a domain-specific evaluation set to know whether it actually worked, and the MLOps loop that keeps
the fine-tuned model current as the taxonomy itself evolves.

## LoRA/QLoRA fine-tuning, applied here rather than re-derived

Course 11's LoRA chapter (`02-instruction-tuning-and-llm-finetuning.md`) covers the mechanics of
low-rank adaptation in depth: freezing the base model's weight matrices, learning a low-rank update
`ΔW = B·A` on top of them instead of updating the full matrix, and why that gets you the large majority
of full fine-tuning's benefit at a small fraction of the trainable-parameter count and GPU memory
footprint. QLoRA extends that further by keeping the frozen base weights quantized during training,
cutting the memory needed to even load the base model. That mechanism applies to Mixtral exactly as
described there — this chapter doesn't re-derive it, it applies it to this platform's specific task.

**What's different here from course 11's application:** course 11 fine-tunes LLaMA 3 for *generation*
tasks (drafting ICF/PLPS/SOC content in a target format and register). This platform's core tasks are
different in kind — **classification** (which regulatory-taxonomy category does this clause or filing
belong to) and **extraction** (pull specific structured fields — effective date, jurisdiction, affected
business line, obligation type — out of unstructured regulatory or policy text). That doesn't change the
fine-tuning mechanism, but it does change what the training data looks like and what "good" means:

```json
{
  "instruction": "Classify the following regulatory filing excerpt against the firm's internal policy taxonomy. Return the single best-matching taxonomy code and extract the effective date and affected business line as structured fields.",
  "input": "Effective 1 March 2027, firms must implement enhanced customer due diligence procedures for correspondent banking relationships involving jurisdictions on the enhanced monitoring list...",
  "output": {
    "taxonomy_code": "AML-CDD-CORR-BANKING",
    "effective_date": "2027-03-01",
    "affected_business_line": "Correspondent Banking"
  }
}
```

Because there's one Mixtral base model but the taxonomy has many top-level categories (AML, sanctions,
capital markets conduct, data privacy, and so on, in a real client taxonomy), the same "one frozen base
model, multiple swappable LoRA adapters" pattern from course 11 applies directly: a shared adapter
covering general taxonomy classification, with the option of category-specific adapters if a particular
category's classification accuracy needs targeted improvement beyond what the general adapter achieves
— trained and swapped independently, without needing to re-host a full separate copy of the ~47B-
parameter base model per category.

**QLoRA's relevance here specifically:** because Mixtral's sparse MoE structure means only ~13B
parameters are active per forward pass (Chapter 1) but the full ~47B still has to be loaded into memory
during training (every expert needs to be present even though only 2-of-8 fire per token), keeping the
frozen base weights in a quantized representation during fine-tuning is a meaningful, not marginal,
memory saving — it's the difference between fine-tuning being practical on a moderate GPU allocation
versus requiring a much larger dedicated training cluster.

## A domain-specific evaluation set

Fine-tuning without a rigorous domain evaluation set is fine-tuning you can't actually validate — this
is the direct analog of course 4's evaluation-harness discipline, applied to this platform's specific
task shape rather than RAG faithfulness/relevance. Two evaluation dimensions matter most:

- **Regulatory-clause classification accuracy.** A held-out set of clauses/filings, each hand-labeled
  with the correct taxonomy code by a domain expert (compliance/legal reviewer, not an engineer),
  covering every top-level taxonomy category and deliberately over-sampling the categories with the
  fewest natural training examples and the categories most prone to ambiguity (adjacent categories that
  are easy to confuse, e.g., a filing that touches both AML and sanctions requirements). Reported as
  per-category accuracy/F1, not just an aggregate number — an aggregate accuracy figure can hide a
  category that's performing badly if it's a small fraction of the overall evaluation set, exactly the
  kind of blind spot Chapter 7's taxonomy-drift design exists to catch on an ongoing basis.
- **Extraction field accuracy.** For each extracted structured field (effective date, jurisdiction,
  affected business line, obligation type), exact-match or normalized-match accuracy against the
  hand-labeled ground truth, scored per field rather than only as "the whole extraction was correct or
  wrong" — a wrong effective date on an otherwise-correct extraction is a different, and often more
  operationally dangerous, failure than a wrong taxonomy code, since a downstream compliance workflow
  might silently schedule the wrong deadline off a wrong extracted date.

This evaluation set is also the mechanism that catches the quantization-quality-degradation failure mode
from Chapter 4 and the taxonomy-drift failure mode from Chapter 7 — it's not a one-time fine-tuning
gate, it's a standing artifact re-run any time the base model, the quantization configuration, or the
fine-tuned adapter changes.

## The MLOps loop for retraining as the taxonomy evolves

The client's regulatory taxonomy is not static — new categories get added as new regulation emerges,
existing categories get split, merged, or deprecated as internal policy is reorganized. A fine-tuned
adapter trained against yesterday's taxonomy silently degrades against today's taxonomy the moment that
taxonomy changes, and Chapter 7 covers the detection and versioning design for exactly that problem in
depth. This chapter's scope is the retraining loop itself:

1. **Taxonomy change lands** (new category added, category deprecated/merged) — recorded as a new
   taxonomy version, per Chapter 7's versioning design.
2. **Training data updated**: new/changed categories need representative labeled examples — ideally
   real historical documents re-labeled under the new taxonomy version by a domain reviewer, plus
   synthetic augmentation (the same "historical pairs plus synthetic augmentation for underrepresented
   cases" pattern course 11 uses for ICF/PLPS/SOC data) for categories too new to have accumulated real
   examples yet.
3. **LoRA adapter retrained** against the updated dataset — cheap enough, relative to a full retraining
   run, that this can realistically happen on a cadence tied to taxonomy-change events rather than a
   slow, infrequent full-retrain schedule.
4. **Evaluated against the updated domain evaluation set** (also updated to reflect the new taxonomy
   version) before promotion — the new adapter has to clear both "does it correctly classify the new/
   changed categories" and "did it regress on the categories that didn't change," since a naive retrain
   focused only on the new categories can quietly hurt performance on unrelated ones.
5. **Registered with an explicit taxonomy-version tag** (Chapter 6's provenance tracking, Chapter 7's
   staleness design) so the model registry can always answer "which taxonomy version was this adapter
   trained against" precisely, rather than relying on institutional memory or a training date as a
   fuzzy proxy.

## Tying It Back

Fine-tuning control (Chapter 2, Reason 2) is only a real advantage if it's exercised responsibly: LoRA/
QLoRA applied to Mixtral exactly as course 11 establishes for LLaMA 3, adapted to this platform's
classification/extraction task shape rather than generation; a domain evaluation set built by actual
compliance/legal reviewers, scored per-category and per-field rather than as one aggregate number; and a
retraining loop that treats taxonomy changes as a first-class trigger, not an afterthought discovered
only when a reviewer catches a misclassification in production — which is exactly the failure mode
Chapter 7 is built to detect before it reaches that point.
