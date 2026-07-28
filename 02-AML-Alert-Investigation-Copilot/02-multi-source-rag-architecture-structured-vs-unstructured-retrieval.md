# 02 — Multi-Source RAG Architecture: Structured vs. Unstructured Retrieval

## Why this chapter exists

This is the architecturally interesting part of the whole project, and the part most worth getting
right in an interview, because it's a genuinely sharp technical point rather than a generic "we used
RAG" answer: **three very different data sources need to feed one narrative, and they should not all
be retrieved the same way.** A lot of RAG implementations reach for a vector index as the default
answer to "how do I get data into an LLM prompt," even when the data is structured, precise, and
already queryable through a normal database call. This chapter makes the case for why that default is
wrong for two of this project's three sources, and right for exactly one of them.

## The three sources, and what kind of data each one actually is

| Source | Data shape | Volume | Precision needs |
|---|---|---|---|
| **KYC profile** | Structured — fixed fields: name, occupation, stated income, risk rating, account tenure, expected activity profile | Small, one record per customer | Needs to be exact — a wrong occupation or risk rating in the narrative is a factual error, not a stylistic one |
| **Transaction history** | Structured, time-series — amount, date, counterparty, channel, currency, per transaction | Can be large — years of history for an active account | Needs to be exact and needs the *right window* — an incomplete or wrongly-filtered lookback silently changes the pattern the narrative describes (Chapter 6 covers a bug in exactly this shape) |
| **Prior case notes** | Unstructured — free-text investigator write-ups from past alerts on this customer | Small to moderate per customer, but semantically varied in phrasing | Needs *relevance*, not exact-match — a prior investigator might describe the same underlying concern in completely different words |

The shape of each source is the whole argument. KYC and transaction data are exactly the kind of data
a relational or document database is built to answer precisely: "give me this customer's risk rating"
or "give me every transaction over $5,000 in the last 90 days" has one correct answer, and a normal
query returns it deterministically. Prior case notes are prose — "give me prior cases relevant to
unusual wire activity" doesn't have one correct answer, it has a *ranked list of plausibly relevant*
documents, which is precisely the problem semantic retrieval is built to solve.

## The sharp point: not everything belongs in a vector index

It's tempting, especially once a RAG pipeline exists for one source, to route everything through it —
embed the KYC record, embed each transaction, embed the case notes, and let similarity search sort it
out. That's the wrong default here, for reasons worth stating precisely rather than just asserting:

- **Embedding-retrieval is fuzzy by construction.** Vector similarity search is a *best-effort*
  mechanism — it returns the chunks most semantically similar to a query, not a guaranteed, exhaustive,
  provably-correct answer. For "what is this customer's stated occupation," fuzzy best-effort retrieval
  is strictly worse than a direct `SELECT occupation FROM kyc_profiles WHERE customer_id = ?` — the
  direct query is deterministic, cheaper, faster, and trivially auditable (you can point at the exact
  query and the exact row it returned). There is no version of embedding a KYC record that improves on
  that.
- **Auditability matters more here than almost anywhere else in this curriculum.** Chapter 7 covers
  regulatory-examination requirements in depth, but the short version: if a regulator asks "where did
  this narrative's claim about the customer's transaction volume come from," the answer needs to be "a
  specific, replayable query against the transaction database, filtered on these exact parameters" —
  not "the top-k most similar chunks an embedding model happened to retrieve." A structured query is a
  reproducible fact; a semantic-retrieval result is a probabilistic one, and probabilistic provenance is
  a much weaker answer to give a regulator.
- **Transaction history has a completeness problem a vector index doesn't solve.** "Top-k similar
  transactions" is close to a meaningless query — a narrative needs *the complete, correctly-windowed
  set* of transactions in the relevant period, not the k transactions that happen to embed closest to
  some query vector. This is a case where retrieval quality isn't about relevance ranking at all; it's
  about correctness of a deterministic filter (date range, amount threshold, account ID).
- **Prior case notes are the genuine exception, and it's worth being precise about why.** Case notes are
  free text, written by different investigators over time, describing similar underlying concerns in
  different words ("elevated inbound wire activity" vs. "unusual receipt pattern from overseas
  counterparties" could both describe the same customer behavior). There is no fixed schema to query
  against — the whole point is finding *semantically* related prior context, which is exactly what
  embedding-based retrieval is designed for and structured queries cannot do at all (you can't `SELECT`
  your way to "notes that describe a similar situation in different words").

## Design: a structured query tool for KYC and transactions

Rather than a vector index, KYC and transaction data are exposed to the generation step as a
**function-calling tool** — the same pattern Course 1 covers for the Text-to-Math agent (Chapter 5 of
that course), applied here to data lookups instead of arithmetic. The model (or, more precisely, the
orchestration layer around it) calls a small set of well-defined, parameterized functions:

```python
# Illustrative tool surface exposed to the narrative-generation step
def get_kyc_profile(customer_id: str) -> dict:
    """Deterministic lookup -- one row, exact fields, no ranking involved."""
    ...

def get_transaction_history(
    customer_id: str,
    start_date: str,
    end_date: str,
    min_amount: float | None = None,
) -> list[dict]:
    """Deterministic, filtered query against core banking transaction data.
    The lookback window is an explicit, auditable parameter -- not an
    implicit default buried in a prompt (see Chapter 6's truncation bug)."""
    ...

def get_alert_trigger_detail(alert_id: str) -> dict:
    """The specific rule(s) fired and the specific transaction(s) that
    triggered them -- pulled directly from the transaction-monitoring
    system's own alert record, not reconstructed."""
    ...
```

Every one of these is precise, replayable, and independently testable — you can call
`get_transaction_history` directly and assert the result matches what's in the source system, which is
a much stronger correctness guarantee than "does the retrieved chunk look relevant." This is also
where the citation requirement in Chapter 3 gets its teeth: because each fact came from a specific,
parameterized function call, the narrative can cite it precisely ("Transaction TXN-88213, per
`get_transaction_history`") rather than vaguely.

## Design: Azure AI Search for prior case notes

Prior case notes go through the RAG pattern Course 1 already establishes: chunked, embedded, and
indexed in Azure AI Search, retrieved via semantic or hybrid (vector + keyword) search at generation
time. A few choices specific to this source:

- **Chunking strategy.** Case notes are chunked **per case, not by a fixed token window**, where
  practical — a full prior narrative (or its Red Flags / Recommendation sections specifically) is more
  useful as a coherent retrieved unit than an arbitrary 500-token slice that might cut a red flag in
  half. Where a single case note is unusually long, a secondary section-aware split (using the same
  six-section structure from Chapter 1/3) keeps chunks semantically coherent rather than arbitrary.
- **Hybrid search over pure vector search.** Investigator phrasing often includes exact terms worth
  matching literally — a specific counterparty name, a specific rule ID, a specific jurisdiction — that
  keyword search catches reliably and pure embedding similarity can miss or under-weight. Azure AI
  Search's hybrid mode (vector + BM25-style keyword, with reciprocal rank fusion) is the right default
  here for the same reason Course 8's HybridSearch RAG chapter argues it in a different domain.
  Retrieval is filtered to the current customer's prior cases first (a metadata filter on
  `customer_id`), with semantic ranking applied within that filtered set — this is a case where a
  structured pre-filter and semantic search compose together rather than compete.
- **What doesn't get embedded.** Only the narrative text itself is embedded — structured metadata
  (case ID, customer ID, date, outcome/recommendation) rides alongside each chunk as filterable fields,
  not as text baked into the embedding, so a query can combine "semantically similar to this alert" with
  a hard filter on "this customer" or "outcome = escalated" without degrading embedding quality with
  metadata noise.

## Assembling both retrieval results into one generation prompt

The generation step (Chapter 3) receives both retrieval mechanisms' output in one prompt, clearly
delineated by source and provenance so the model can (and is instructed to) cite each fact precisely:

```python
# Illustrative prompt assembly
prompt_context = f"""
== CUSTOMER KYC PROFILE (source: get_kyc_profile, customer_id={customer_id}) ==
{kyc_profile_json}

== TRANSACTION HISTORY (source: get_transaction_history, {start_date} to {end_date}) ==
{transaction_history_json}

== ALERT TRIGGER DETAIL (source: get_alert_trigger_detail, alert_id={alert_id}) ==
{alert_detail_json}

== RETRIEVED PRIOR CASE NOTES (source: Azure AI Search, semantic/hybrid, top-5) ==
{retrieved_case_note_chunks_with_case_ids}
"""
```

The structured data arrives as exact JSON with an explicit source function named; the retrieved case
notes arrive as text chunks each tagged with their originating case ID. Both are grounding material for
the same generation call, but the model is instructed (Chapter 3) to cite them differently — a
structured fact by its field/transaction ID, a case-note-derived claim by its case ID — because the
*confidence* behind each is genuinely different: one is a database fact, the other is "a prior
investigator's semantically similar write-up," and conflating the two in how they're cited would hide
a real difference in how verifiable each claim actually is.

## Tying It Back

The one-sentence version of this chapter, worth having ready: *"KYC and transaction data go through a
structured, function-calling tool because they're precise, exact-match data where a vector index would
trade away correctness for a retrieval mechanism they don't need; prior case notes go through Azure AI
Search's semantic/hybrid retrieval because they're free text where fuzzy relevance matching is exactly
the right tool — and both feed the same generation prompt, tagged clearly enough by source that the
model can cite each fact back to something verifiable."* Chapter 3 picks up from here: how that
combined context gets turned into a structured, six-section narrative with per-claim citations as the
anti-hallucination defense.
