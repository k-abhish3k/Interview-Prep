# 02 — Instruction Tuning & LLM Fine-Tuning

## Why this chapter matters

The resume bullet says "required Instruction-based content generation for various modules of the
platform," and the skills list names both **LLAMA 3** and **LLM Finetuning** as separate items. That
pairing is not a coincidence — it points at a specific engineering decision: rather than prompting a
generic base model and hoping it produces ICF-shaped or SOC-shaped output every time, the platform
needed a model that reliably *follows instructions* in a fairly narrow, high-stakes domain. This
chapter explains what "instruction-tuned" actually means, how you'd build the data to get there, and
why parameter-efficient fine-tuning (LoRA/QLoRA) is the practical way to specialize a model like
LLaMA 3 without paying full fine-tuning cost — the concept every interviewer wants you to be able to
reason about from first principles, not just name-drop.

## Base model vs. instruction-tuned model

A **base** (pretrained) LLM is trained purely to predict the next token over a huge corpus of raw
text. It has broad knowledge and strong language modeling ability, but no particular bias toward
"answering the user's request helpfully and in the requested format" — ask it a question and it may
just as easily continue the text as if it were writing a related passage, rather than answering
directly. It has learned *what plausible text looks like*, not *what a good response to an
instruction looks like*.

An **instruction-tuned** model (what LLaMA 3's "-Instruct" variants are, and what SFT — supervised
fine-tuning — on instruction data produces) has been further trained on examples that look like
`(instruction, input, desired output)` — so it has learned the *behavior* of following an
instruction and producing output in the requested shape, not just generating plausible continuations.
This is exactly the behavior a regulatory-document platform depends on: given "summarize this
Eligibility Criteria section in plain language, at an 8th-grade reading level, preserving every
inclusion/exclusion condition," you need reliable compliance with *both* the task and the format,
across thousands of sections and section types, not an occasionally-cooperative model.

For this platform, three tiers of "instruction following" apply, cheapest to most involved:

1. **Prompt engineering alone** on an off-the-shelf instruction-tuned model — no training, fastest to
   ship, works reasonably well when the task is close to what the base instruct-tuned model has
   already seen broadly (general summarization, general Q&A).
2. **Parameter-efficient fine-tuning (LoRA/QLoRA)** on top of an instruction-tuned base — moderate
   cost, used when the domain/format is specific enough (ICF's exact structure, SOC's exact
   change-record schema, the register and terminology conventions of clinical writing) that prompting
   alone doesn't reliably hold the format or vocabulary.
3. **Full fine-tuning** — updates every model weight, most expensive in compute and data, generally
   reserved for cases where PEFT genuinely can't capture the needed behavior change, which is rare in
   practice for a well-scoped domain-adaptation task like this one.

For a project like this, (2) is the realistic sweet spot, and it's the one worth being able to
explain in depth.

## Constructing an instruction dataset

Instruction tuning data is a set of `(instruction, input, output)` triples — sometimes called
"instruction/input/output" or "prompt/completion" pairs. For this platform, each triple represents
one desired behavior of the model:

```json
{
  "instruction": "Summarize the following clinical trial protocol section into a Plain Language Protocol Synopsis (PLPS) entry. Preserve every eligibility condition. Write at an 8th-grade reading level. Do not introduce any claim not present in the source text.",
  "input": "3.1 Inclusion Criteria\n1. Adults aged 18-75 with confirmed diagnosis of...\n2. Able to provide written informed consent...",
  "output": "To join this study, you must: be an adult between 18 and 75 years old and have been diagnosed with... You must also be able to read and sign a consent form..."
}
```

Building a dataset like this for a real platform typically draws on:

- **Historical document pairs.** If the organization has produced ICF/PLPS/SOC documents by hand for
  past trials, each (Protocol section, corresponding human-written ICF/PLPS/SOC section) pair is a
  natural training example — this is the highest-value data source because it's real, human-approved
  output. Because this platform served two competing pharma clients, Eli Lilly and AstraZeneca,
  historical document pairs and any resulting fine-tuning data were kept per-client, never pooled
  across clients, consistent with the isolation requirements in `00-README.md`.
- **Synthetic augmentation.** For section types or edge cases underrepresented in the historical
  archive (e.g., rare amendment types for SOC), a strong general-purpose model can draft candidate
  training examples that a human reviewer then corrects — this is a standard, practical way to expand
  coverage without needing thousands of real historical examples for every section type.
- **Per-module, per-section-type structure.** Because ICF, PLPS, and SOC each have distinct formats
  (patient-facing narrative vs. structured change log), the instruction dataset is usually organized
  as separate instruction "families" — one for each module/section-type combination — rather than one
  undifferentiated pile of examples, so the model reliably learns each format rather than blending
  them.

The notebook `02_instruction_finetuning_demo_lora.ipynb` builds a small synthetic dataset in exactly
this shape for a "summarize this protocol section in plain language" task, to make the data format
concrete.

## Parameter-efficient fine-tuning: LoRA and QLoRA

Full fine-tuning of a model the size of LLaMA 3 means updating every one of its (tens of) billions of
parameters — that requires storing gradients and optimizer state for the full parameter count, which
is expensive in GPU memory and compute, and produces a full new copy of the model per fine-tuned
variant. For a platform that needs several specialized behaviors (ICF style, PLPS style, SOC style),
that cost multiplies.

**LoRA (Low-Rank Adaptation)** is built on an empirical observation: the *change* needed in a weight
matrix to adapt a pretrained model to a new task tends to have low "intrinsic rank" — you don't need
a full-rank update to capture it. So instead of updating a weight matrix `W` directly, LoRA freezes
`W` and learns a low-rank decomposition `ΔW = B·A`, where `B` and `A` are much smaller matrices (rank
`r`, typically 4-64, versus the full dimension which can be in the thousands). The effective forward
pass becomes:

```
output = x · (W + B·A)
```

Only `A` and `B` are trained; `W` never changes. If `W` is `d × d`, a full update has `d²` trainable
parameters; a rank-`r` LoRA update has `2·d·r` — for `d` in the thousands and `r` in the tens, that's
often **less than 1% of the parameters** of a full update. The from-scratch numpy demo in
`02_instruction_finetuning_demo_lora.ipynb` makes this concrete: it constructs a weight matrix, shows
what a full-rank update looks like, then shows the same adaptation expressed as `B·A` with a small
rank, and directly counts trainable parameters both ways.

**QLoRA** extends this further by keeping the frozen base model weights in a quantized (lower
bit-width, typically 4-bit) representation during training, cutting the GPU memory needed to even
*load* the base model — the low-rank adapter matrices themselves are still trained in higher
precision. This is what makes fine-tuning a model the size of LLaMA 3 practical on a modest number of
GPUs rather than requiring a large dedicated training cluster.

Why this matters for this specific platform:

- **Multiple specialized behaviors from one base model.** Because LoRA produces small adapter weights
  (megabytes, not gigabytes) rather than a full model copy, it's practical to train a *separate LoRA
  adapter per module/section-type* (an ICF adapter, a PLPS adapter, an SOC adapter) and swap adapters
  at inference time on top of the same frozen LLaMA 3 base — versus needing to host N full fine-tuned
  models.
- **Faster iteration.** Regulatory templates and terminology conventions do shift (new sponsor,
  updated ICH guidance). Retraining a LoRA adapter on updated examples is far cheaper than a full
  fine-tuning run, which matters for a platform that needs to stay current.
- **Lower risk of catastrophic forgetting.** Because the base weights are frozen, the model's general
  language ability and broad clinical knowledge (learned during pretraining) are preserved — you're
  adding a narrow behavioral adjustment on top, not overwriting the base model's competence.

## Where prompt engineering still does the heavy lifting

Fine-tuning and prompt engineering are not either/or — in practice, the fine-tuned/instruction-tuned
model is still driven by carefully designed prompts per section type (Chapter 05 goes deep on this).
Fine-tuning teaches the model the *general shape and register* of ICF/PLPS/SOC content; the prompt at
inference time supplies the *specific source section* and *specific instructions* for that call
(reading level, which criteria to preserve, what format the output schema requires). Treating
fine-tuning as "prompting you only have to do once, baked into the weights" is a useful mental model
for explaining the division of labor in an interview.

## Tying it back

This chapter is the direct answer to "how did you use LLaMA 3 and LLM fine-tuning on this project."
The honest, defensible answer is: an instruction-tuned LLaMA 3 base, specialized per module/section-
type via LoRA (or QLoRA, if GPU memory was a binding constraint) on an instruction dataset built from
historical ICF/PLPS/SOC document pairs plus synthetic augmentation for underrepresented section
types — chosen over full fine-tuning because the task is a narrow behavioral adaptation (format,
register, terminology discipline) on top of a model that already has strong general language and
domain knowledge, not a wholesale change to what the model knows.
