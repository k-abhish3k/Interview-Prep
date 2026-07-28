# 05 — Building Instruction-Based Generation Pipelines

## Why this chapter matters

This is the chapter where every earlier concept comes together into the thing the resume bullet
actually names: "Instruction-based content generation for various modules of the platform." Chapters
01-04 covered the ingredients (structured input, a fine-tuned instruction-following model, a graph
representation for comparison). This chapter covers how you assemble those ingredients into a
*controlled* generation pipeline for regulated content — one where "the model produced plausible
text" is never good enough on its own, and "the output is traceable, checkable, and reviewed" is the
actual bar.

## The core design principle: generation is a draft, not a deliverable

Every design choice in this chapter follows from one constraint: ICF, PLPS, and SOC content directly
affects patient understanding and regulatory submissions, so an ungrounded or malformed generation
is not a minor bug — it's a compliance and patient-safety risk. That means the pipeline's job isn't
"generate the best possible text," it's "generate the best possible *draft*, constrained tightly
enough that a human reviewer can verify it quickly and confidently." Every technique below serves that
goal.

## Prompt templates per document-section type

ICF, PLPS, and SOC sections aren't interchangeable — they have different audiences, formats, and
constraints, so a single generic "summarize this" prompt doesn't hold up across all of them. A
practical pipeline maintains a **template per section type**, each encoding the constraints specific
to that content:

```python
ICF_ELIGIBILITY_TEMPLATE = """You are drafting the "Who can take part" section of an Informed Consent Form.

Source (Protocol Eligibility Criteria, verbatim):
{source_text}

Instructions:
- Write at an 8th-grade reading level.
- Preserve every inclusion and exclusion condition present in the source — do not omit any.
- Do not state anything not present in the source text.
- Do not use clinical abbreviations; spell out terms in plain language.
- Output as a bulleted list, one condition per bullet.
"""

SOC_ENTRY_TEMPLATE = """You are drafting one entry of a Summary of Changes (SOC) document.

Previous version (Section {section_number}):
{previous_text}

Current version (Section {section_number}):
{current_text}

Instructions:
- Identify the substantive change(s) only; ignore purely stylistic rewording with no meaning change.
- Output valid JSON: {{"section": "...", "change_type": "added|removed|modified", "summary": "...", "rationale_if_stated": "..."}}
- If no substantive change is present, output {{"change_type": "none"}}.
"""
```

Notice the shape of both: source text is passed in explicitly (never assumed to be "in the model's
memory" from training), the instructions are specific about both content constraints (preserve every
condition; ignore stylistic-only changes) and format constraints (bulleted list; JSON schema), and
each template is scoped to exactly one section type rather than trying to be general-purpose. This
per-section-type template design is what Chapter 01's structuring stage exists to feed — a
`Requirement`/section object with a known type is what lets the pipeline pick the right template
automatically.

## Structured, schema-constrained output

Free-text generation is hard to programmatically verify and hard to feed into downstream systems
(the graph model in Chapter 04, a document-assembly template, an audit log). For section types with a
well-defined structure — SOC entries are the clearest example — the pipeline should ask for, and
validate, structured output:

```python
import json

def generate_soc_entry(llm_call, previous_text, current_text, section_number):
    raw_output = llm_call(SOC_ENTRY_TEMPLATE.format(
        previous_text=previous_text, current_text=current_text, section_number=section_number
    ))
    try:
        parsed = json.loads(raw_output)
    except json.JSONDecodeError:
        return {"status": "needs_review", "reason": "model did not return valid JSON", "raw": raw_output}

    required_keys = {"section", "change_type", "summary"}
    if parsed.get("change_type") != "none" and not required_keys.issubset(parsed):
        return {"status": "needs_review", "reason": "missing required fields", "raw": parsed}

    return {"status": "ok", "data": parsed}
```

Anything that fails to parse or fails schema validation is routed straight to manual review rather
than silently discarded or auto-corrected — a malformed output is itself a signal that the generation
needs a human look, not something to paper over programmatically. In production, this is typically
backed by function-calling / constrained-decoding features on the serving side (Chapter 03) that
restrict the model's output distribution to valid JSON at generation time, rather than relying purely
on post-hoc parsing and retry.

## Factual grounding: preventing hallucinated clinical claims

This is the single highest-stakes technical problem in the whole pipeline: **how do you stop the
model from stating something about the trial that isn't actually in the source Protocol?** A few
layered defenses, used together rather than any one alone:

1. **Retrieval-scoped generation.** Every generation call receives only the relevant, structured
   source section (from Chapter 01's parsing stage) — never "the whole Protocol, figure out the
   relevant part yourself." Narrowing the input narrows what the model has available to draw from,
   which structurally reduces (though doesn't eliminate) the surface for fabrication.
2. **Explicit grounding instructions.** The prompt directly instructs the model not to introduce
   claims absent from the source, and — for higher-stakes sections like eligibility criteria or safety
   information — to prefer omission over invention when uncertain. This alone is not sufficient (models
   don't reliably self-police), but it measurably reduces the error rate.
3. **Post-generation grounding checks.** After generation, run an automated check that extracts key
   entities/terms/claims from the generated text and verifies each appears (or has a clear paraphrase)
   in the source section. A simple, interview-explainable version of this: extract key noun phrases
   or clinical terms from the generated output, check for their presence (exact or fuzzy) in the
   source text, and flag anything that doesn't match for reviewer attention. The notebook
   `04_protocol_comparison_pipeline.ipynb` builds exactly this kind of check — pure Python, no model
   calls, operating on a mock-generated PLPS paragraph against its source section — to make the
   mechanism concrete.
4. **A second-pass verifier call**, in more sophisticated pipelines: a separate model prompt whose
   only job is "given this source and this generated text, list any statement in the generated text
   not supported by the source" — using the model itself as a checker, with a different, narrower
   objective than the generation call had. This catches subtler unsupported claims that simple term-
   matching misses, at the cost of an extra inference call.
5. **Confidence/flag surfacing to the reviewer**, not silent correction. None of the above should
   auto-edit the output to "fix" an ungrounded claim — that's still the model guessing. The check's job
   is to flag it clearly so the human reviewer knows exactly where to look first.

## The human-review and approval gate

Every generated section — regardless of how clean the grounding check comes back — passes through a
review and approval stage before it's used in an ICF, PLPS, SOC, or comparison output. The design
goals for that gate:

- **Reviewer sees the source alongside the generation**, ideally with the specific source sentences
  the grounding check matched (or failed to match) highlighted — turning "re-read the whole Protocol"
  into "check these three highlighted claims against these three highlighted source sentences."
- **Every review action is logged**: who reviewed, what was approved/edited/rejected, and when — this
  is the traceability regulators and sponsors expect, and it's the natural answer to a system-design
  question about auditability (see the Q&A bank).
- **Edits feed back into the fine-tuning data.** A reviewer's correction to a generated ICF paragraph
  is exactly the kind of real, human-approved example that improves the next round of LoRA fine-tuning
  (Chapter 02) — closing the loop between production usage and model improvement, rather than treating
  reviewed output as a dead end.
- **Nothing generated is published, submitted, or shown to a patient without explicit approval.** This
  is a hard gate, not a configurable setting — the whole value proposition of the platform is "faster
  first drafts," not "faster publication," and collapsing that distinction is the single biggest risk
  to get wrong when describing this project in an interview. In production, this reviewer is a real
  regulatory-affairs professional at Eli Lilly or AstraZeneca, using the platform daily — which is
  exactly why this gate is treated as non-negotiable rather than a configurable pipeline step.

## Tying it back

If asked to describe the generation pipeline end to end: a structured source section (Chapter 01)
feeds a section-type-specific prompt template running against a LoRA-fine-tuned, instruction-tuned
LLaMA 3 (Chapter 02, 03); output is validated against a schema where the section type calls for
structure (SOC); a grounding check verifies generated claims trace back to the source; and every
result — clean or flagged — goes to a human reviewer whose approval is the only path to publication.
That's the full loop the resume bullet is describing, and it's also the strongest possible answer to
"how do you build GenAI systems responsibly in a regulated domain": not by trusting the model more,
but by constraining, checking, and gating it at every stage where a mistake would actually matter.
