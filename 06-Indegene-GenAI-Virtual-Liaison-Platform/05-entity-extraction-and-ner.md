# 05 — Entity Extraction & NER

## Why this chapter matters

"Entity-extraction-based Project demand capture" is the feature that turns free-text client requests
("I need this deck localized into Japanese and Korean by end of quarter, priority region APAC") into
a structured, actionable record the downstream project-creation system can consume. This is a classic
NLP problem — Named Entity Recognition (NER) — but the interesting interview material is *how* you'd
solve it for a task like this: rule-based, statistical/spaCy, or LLM-based structured extraction, and
why the LLM approach usually wins for open-ended demand capture even though it's the newest and least
"traditional NLP" of the three.

## Three Approaches to Entity Extraction

### 1. Rule-Based Extraction (Regex / Pattern Matching)

The simplest approach: hand-written patterns that match known formats.

```python
import re

DEADLINE_PATTERN = re.compile(
    r"\b(?:by|before|due)\s+([A-Za-z]+\s+\d{1,2}(?:,?\s+\d{4})?|end of \w+|Q[1-4]\s*\d{0,4})",
    re.IGNORECASE,
)
LANGUAGE_PATTERN = re.compile(
    r"\b(Japanese|Korean|French|German|Spanish|Mandarin|Portuguese)\b", re.IGNORECASE
)

text = "I need this deck localized into Japanese and Korean by end of quarter."
deadline = DEADLINE_PATTERN.search(text)
languages = LANGUAGE_PATTERN.findall(text)
```

Rule-based extraction is fast, free, fully deterministic, and easy to unit test — but it only catches
what you explicitly anticipated. It breaks the moment a client phrases a request in a way the patterns
didn't cover ("I'll need the JP and KR versions" instead of "Japanese and Korean"), and every new
entity type or phrasing variant means writing and maintaining more regex. It's a reasonable
**first-pass filter** or fallback, not a complete solution for open-ended client language.

### 2. Statistical/Neural NER (spaCy)

spaCy ships pretrained NER models that recognize general-purpose entity types (`PERSON`, `ORG`, `GPE`
— geopolitical entity, `DATE`, `LANGUAGE` in some pipelines) out of the box, using a trained
statistical/neural model rather than hand-written rules — so it generalizes better to phrasing the
rules never saw.

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("I need this deck localized into Japanese and Korean by end of quarter, region APAC.")
for ent in doc.ents:
    print(ent.text, ent.label_)
# e.g. "Japanese" NORP, "Korean" NORP, "APAC" ORG/GPE (model-dependent)
```

spaCy's out-of-the-box model is trained on general text, not project-demand language, so it recognizes
generic categories well but has no built-in concept of "asset type" or "priority region" as domain-
specific fields — it would need **fine-tuning on labeled project-request examples**, or a custom
`EntityRuler` layered on top of the statistical model, to reliably extract Indegene-specific entity
types. That's a real engineering investment (labeled training data, a training/eval loop, ongoing
maintenance as request phrasing evolves) — worth naming as the trade-off against approach 3.

### 3. LLM-Based Structured Extraction (Function Calling / Structured Output)

Rather than training a custom NER model, prompt an LLM to extract entities directly into a
predefined schema, using structured output (function calling / JSON mode / a Pydantic-backed parser).
This is the approach that fits an open-ended, evolving set of project-request fields best, because
"training data" for a new field is just... adding it to the schema and a couple of few-shot examples,
not collecting and labeling hundreds of new examples.

```python
from pydantic import BaseModel, Field
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate

class ProjectDemandRequest(BaseModel):
    asset_type: str = Field(description="Type of deliverable requested, e.g. 'promotional deck'")
    languages: list[str] = Field(description="Target languages for localization")
    region: str | None = Field(default=None, description="Priority region, if mentioned")
    deadline: str | None = Field(default=None, description="Requested deadline, if mentioned")
    reference_project: str | None = Field(
        default=None, description="A prior project referenced as similar, if mentioned"
    )

parser = PydanticOutputParser(pydantic_object=ProjectDemandRequest)

extraction_prompt = ChatPromptTemplate.from_messages([
    ("system",
     "Extract a structured project request from the client's message. "
     "Leave fields null if not mentioned. Do not invent values.\n{format_instructions}"),
    ("human", "{client_message}"),
])

extraction_chain = extraction_prompt | llm | parser
request = extraction_chain.invoke({
    "client_message": "I need this deck localized into Japanese and Korean by end of quarter, "
                       "similar to what we did for Project Atlas.",
    "format_instructions": parser.get_format_instructions(),
})
# ProjectDemandRequest(asset_type="promotional deck", languages=["Japanese", "Korean"],
#                       region=None, deadline="end of quarter", reference_project="Project Atlas")
```

This handles paraphrase, multi-field extraction in a single pass, and — since translation/language
detection already sits upstream in this platform's architecture (Chapter 0's diagram) — works
naturally across languages once the client's message is normalized to a common working language, with
no separate per-language NER model needed.

## Choosing an Approach for This Feature

| | Rule-based | spaCy (statistical) | LLM structured output |
|---|---|---|---|
| Handles novel phrasing | No | Partially | Yes |
| Needs labeled training data | No | Yes, to customize | No (few-shot / schema only) |
| Cost per extraction | ~free | Low (local inference) | LLM API call cost |
| Latency | Fastest | Fast | Slowest (network + generation) |
| Handles multi-lingual input | No | Needs per-language model | Yes (LLM handles it directly) |
| Easiest to extend with a new field | No — new regex | Medium — retrain/re-rule | Easiest — edit schema/prompt |

For an evolving set of project-request fields captured from free-form, multilingual client messages,
**LLM-based structured extraction is the right default**, with rule-based regex kept as a cheap
validation/fallback layer — e.g., double-checking that an extracted "deadline" string actually
parses as a date, or catching obviously malformed output before it reaches the project-creation
system. That combination (LLM does the heavy lifting of understanding open-ended language; rules do
cheap, deterministic sanity-checking of the output) is a pattern worth naming explicitly, since
"trusting the LLM's output blindly" is exactly the kind of gap an interviewer will probe for with "how
did you validate the extracted entities before creating a project record?"

## Handling Uncertainty and Missing Fields

A production version of this feature can't force every field to be present — clients often omit the
deadline or don't specify a region. The schema above models this with optional (`| None`) fields
rather than requiring the LLM to invent a value, and the system prompt explicitly instructs "do not
invent values" — a direct defense against a well-known LLM structured-extraction failure mode:
silently hallucinating a plausible-looking value for a field that was never actually mentioned. When a
required field for project creation is missing, the natural fallback in a conversational platform is
to have the assistant ask a clarifying follow-up ("by when do you need this?") rather than guessing —
which is also a good place to mention that entity extraction here isn't a one-shot batch process, it's
one turn in a multi-turn conversation, tying forward into Chapter 7's LangGraph state management.

## Tying It Back

When asked "how did you turn a free-text client request into a structured project record," the
answer is LLM-based structured extraction with a Pydantic schema defining the target fields
(asset type, languages, region, deadline, referenced prior project), explicit instructions against
inventing unmentioned values, and a rule-based validation pass on the output before it reaches
project creation — with the choice of LLM extraction over training a custom spaCy NER model justified
by the open-ended, multilingual, evolving nature of client requests versus the fixed-vocabulary,
labeled-data-hungry nature of a trained statistical NER model.
`notebooks/05_ner_entity_extraction.ipynb` demonstrates both the rule-based regex approach and a
spaCy NER example (with a graceful fallback if spaCy/its model isn't installed) on a synthetic project
request sentence.
