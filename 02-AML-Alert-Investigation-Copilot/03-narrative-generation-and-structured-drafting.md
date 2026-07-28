# 03 — Narrative Generation and Structured Drafting

## Why this chapter exists

Chapter 2 covers how the right data gets into the prompt. This chapter covers what happens next: how
that context turns into an actual case narrative that an investigator would recognize as "ready," in
the exact structure compliance expects, with every factual claim traceable back to something a
compliance officer can verify. This is not a conversational chat response — nobody wants the model to
"discuss" an AML alert with a user turn by turn. The output is a **document**: fixed structure, fixed
sections, machine-checkable completeness, and a citation on every material claim.

This chapter deliberately does not re-explain structured-output prompting or format drift from first
principles — **Course 1's Chapter 1 ("Structured Output Prompting" and "Format Drift, In Depth")
already covers that ground in full**, and everything there transfers directly. This chapter applies it
to this project's specific schema and adds the one requirement that's genuinely new here: per-claim
citation as a compliance control, not just a nice-to-have.

## The six-section schema, as a structured-output contract

Chapter 1 defined the six sections a case narrative needs. Here, that structure isn't just a stylistic
convention the model is asked nicely to follow — it's enforced as a schema, the same way Course 1
argues native structured-output/function-calling modes are more reliable than prompt-engineered JSON
formatting (Course 1, Chapter 1):

```python
from pydantic import BaseModel, Field

class Citation(BaseModel):
    source_type: str  # "kyc_field" | "transaction_id" | "prior_case_id"
    source_id: str     # e.g. "occupation", "TXN-88213", "CASE-2024-0091"

class CitedClaim(BaseModel):
    text: str
    citations: list[Citation] = Field(
        default_factory=list,
        description="Every factual claim must cite at least one source. "
                     "An empty list here means the claim is unverifiable "
                     "and should not have been generated.",
    )

class CaseNarrative(BaseModel):
    customer_kyc_overview: list[CitedClaim]
    alert_trigger_summary: list[CitedClaim]
    transaction_pattern_analysis: list[CitedClaim]
    historical_prior_case_context: list[CitedClaim]
    red_flags_identified: list[CitedClaim]
    recommendation: str  # "close_false_positive" | "escalate" | "sar_referral"
    recommendation_rationale: list[CitedClaim]
    confidence_by_section: dict[str, float]  # Chapter 4 -- per-section confidence
```

Generating against this schema via Azure OpenAI's native structured-output/function-calling mode
(rather than a prompt-engineered "please respond in this JSON format") means a malformed or incomplete
narrative is a **schema validation failure the orchestration layer catches immediately**, not a subtle
formatting slip that reaches a compliance officer's queue looking plausible. Every section is
guaranteed present (even if a section's list ends up empty — which is itself informative, not a
missing-data bug) because the schema requires the key to exist, closing off the specific "field goes
missing under drift" failure mode Course 1's Format Drift section describes.

One schema decision worth calling out deliberately: **each section is a list of individually-cited
claims, not one prose blob.** That's a stronger anti-hallucination structure than a single narrative
paragraph per section, because it forces the model to attach a citation (or admit it can't) to each
discrete assertion rather than letting one well-cited sentence "cover for" three uncited ones sitting
next to it in the same paragraph.

## Why one narrative call, not a conversational loop

Course 1's chatbot is built around multi-turn conversation — the whole point is a back-and-forth. This
project's generation step is architecturally closer to a **single, well-specified batch call**: given
the assembled context from Chapter 2 (KYC, transaction history, alert detail, retrieved case notes),
produce one complete `CaseNarrative` object. There's no back-and-forth with the model during drafting
— the "conversation," such as it is, happens afterward, between the human reviewer and the draft
(Chapter 4), not between the human and the model in real time. This matters for two practical reasons:
it sidesteps the format-drift-over-many-turns problem Course 1's Chapter 1 describes almost entirely
(there's no long conversation history to dilute the format instruction, because there's exactly one
call), and it means the same generation call can be evaluated, regression-tested, and reproduced
deterministically (low temperature, fixed context) far more easily than a multi-turn dialogue can.

## Prompt structure

```python
SYSTEM_PROMPT = """
You are an AML case-narrative drafting assistant. You draft investigator-ready
case narratives for a compliance officer's review -- you do NOT make the final
determination, and your output is always reviewed before it becomes part of the
official case record.

Rules:
1. Respond ONLY using the CaseNarrative schema provided via structured output.
2. Every claim in every section must be backed by at least one citation to a
   specific KYC field, transaction ID, or prior case ID drawn from the context
   below. If you cannot support a claim with a citation, do not include it.
3. Do not speculate about facts not present in the provided context. If the
   context is insufficient to assess a section, say so explicitly rather than
   inferring.
4. Assign a confidence score (0-1) per section reflecting how directly the
   provided context supports that section's content -- low confidence where
   context was thin (e.g., very few prior cases retrieved), not just where the
   language itself feels uncertain.
5. Your recommendation must follow from the red flags you identified -- do not
   recommend escalation without at least one cited red flag, and do not
   recommend closure while an identified red flag remains unaddressed in your
   own rationale.
"""
```

Rule 2 is the load-bearing one, and it's worth walking through why it works as an anti-hallucination
mechanism rather than just a compliance nicety.

## The citation/grounding requirement as the primary anti-hallucination defense

This is the single most important idea in this chapter, and it ties directly back to Course 1's RAG-
grounding content: an LLM instructed to "only answer from provided context" (Course 1, Chapter 3) is a
*prompt-level* defense — it reduces hallucination but doesn't make it verifiable after the fact. The
citation requirement here goes one step further: it makes every factual claim **independently checkable
against the source data**, which means hallucination isn't just discouraged, it's *detectable*.

Concretely, this enables a verification step that runs *after* generation, before a narrative ever
reaches a compliance officer's queue: for every cited transaction ID, confirm it actually appears in
the `get_transaction_history` result that was in context; for every cited prior case ID, confirm it
actually appears in the retrieved Azure AI Search chunks; for every cited KYC field, confirm the field
name and value match what `get_kyc_profile` actually returned. A citation that doesn't resolve against
the source data it claims to come from is a **hallucinated citation** — a serious finding for a
compliance document, and exactly the failure mode `notebooks/03_citation_grounding_and_hallucination_check.ipynb`
demonstrates concretely, and Chapter 6 covers a real-shaped bug narrative where this check would have
(and eventually did) catch one.

```python
# Illustrative post-generation verification, before the draft reaches review
def verify_citations(narrative: CaseNarrative, source_data: SourceBundle) -> list[str]:
    """Returns a list of citation strings that don't resolve against the
    actual retrieved/queried source data -- i.e. hallucinated citations."""
    broken = []
    for claim in narrative.all_claims():
        for citation in claim.citations:
            if not source_data.resolves(citation):
                broken.append(f"{citation.source_type}:{citation.source_id}")
    return broken
```

A narrative with any unresolved citation is flagged before it ever reaches a human reviewer — either
regenerated automatically with the offending claim removed, or (more conservatively, and the safer
default for a first production version) routed to review with the unresolved citation explicitly
flagged so the reviewer knows exactly which claim to scrutinize rather than trusting the whole document
uniformly. This distinction matters: **citation verification catches "this citation doesn't point to
anything real,"** a mechanical check; it does **not** catch "this citation is real but the claim built
on it is subtly wrong or overstated" — that second, harder problem is exactly why the human review step
in Chapter 4 still exists even after citation verification passes cleanly. Automated grounding checks
and human judgment are complementary controls, not substitutes for each other.

## Per-section confidence, and why it's generated, not just inferred

The schema's `confidence_by_section` field is worth a specific word, because it's easy to conflate with
"the model's own certainty," which is not quite what it means here. It's explicitly instructed (Rule 4
above) to reflect **how well the retrieved/queried context supports that section**, not how fluent or
assertive the model's own language sounds — a section can read confidently while resting on thin
context (e.g., zero prior cases were retrieved for the historical-context section, so there's
genuinely little to say, and the confidence score should say so even if the model still produces a
grammatically confident sentence). This score becomes the input to Chapter 4's review-prioritization —
a compliance officer scrutinizes a low-confidence section more carefully than a high-confidence one
grounded in a dense set of citations.

## Tying It Back

Structured output turns "does the narrative have all the right sections" from a hope into a schema
guarantee, applying Course 1's structured-output and format-drift lessons to a single, well-specified
drafting call rather than a long conversation. The citation requirement is the piece that's genuinely
new to this project: it converts "trust the model's synthesis" into "verify every claim against the
data it's supposed to have come from," which is what makes an automated hallucination check possible at
all, and it's the reason a compliance officer reviewing this narrative (Chapter 4) has something
concrete to check each claim against, rather than a well-written paragraph they have to take on faith.
