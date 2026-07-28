# 03 — HybridSearch RAG

## Why this chapter matters

"HybridSearch RAG" is a named skill on the resume, distinct from plain RAG, and it maps cleanly onto
the two sub-features most likely to suffer from pure dense retrieval: project codenames ("Project
Atlas", "SKU-JP-4471") and the cost catalog. This chapter explains precisely *why* dense retrieval
alone fails on exact identifiers, how sparse retrieval fixes it, and how the two get fused into one
ranking — a question almost guaranteed to come up as "hybrid search vs. pure dense — when does it
matter?"

## Where Pure Dense Retrieval Breaks Down

Chapter 1 covered dense retrieval: embed the query and every chunk, rank by vector similarity.
Embeddings are excellent at capturing *meaning* — "how long for the translation" matching a chunk
about "localization turnaround time" despite no shared words. But embeddings are comparatively weak
at **exact lexical matching** of low-frequency, high-specificity tokens:

- **Project codenames** like "Project Atlas" or internal IDs like "PRJ-2024-0119" are often
  out-of-vocabulary or rare in the embedding model's training data — the model has no strong learned
  representation for a string it's essentially never seen, so semantically-similar-looking but wrong
  codenames can end up embedded close to the right one.
- **Cost-catalog SKUs** ("SKU-LOC-JP-STD", "CAT-4471-B") are effectively arbitrary strings from the
  embedding model's point of view — there's no "meaning" for the model to capture, only an identity
  to match exactly.
- A client asking "what's the price for SKU-4471" needs that literal token found, not something
  semantically adjacent to it.

Keyword-based (sparse) retrieval is the inverse: strong on exact/near-exact term matching, weak on
paraphrase and synonymy ("translation" vs. "localization"). **Hybrid search runs both and combines
them**, so the system gets dense retrieval's semantic recall and sparse retrieval's exact-match
precision at the same time — neither alone covers what the Virtual Liaison's query mix actually needs.

## Sparse Retrieval: BM25

BM25 (Best Matching 25) is the standard modern sparse-retrieval algorithm — a refinement of TF-IDF
that scores how well a document matches a query based on term frequency, inverse document frequency,
and document-length normalization.

- **Term frequency (TF)**: how often a query term appears in a document — more occurrences suggest
  stronger relevance, with diminishing returns (a term appearing 10 times isn't 10x as relevant as
  once).
- **Inverse document frequency (IDF)**: rare terms across the corpus are more informative than common
  ones — "SKU-4471" appearing in a document is far more discriminating than the word "project"
  appearing in it, since "project" appears almost everywhere.
- **Length normalization**: a long document naturally contains more term matches by chance; BM25
  discounts for that so short, precisely-matching chunks aren't unfairly outranked by long, loosely-
  matching ones.

```python
from rank_bm25 import BM25Okapi

tokenized_corpus = [doc.lower().split() for doc in catalog_chunks]
bm25 = BM25Okapi(tokenized_corpus)

query = "price for SKU-4471 japan localization".lower().split()
bm25_scores = bm25.get_scores(query)   # one score per chunk, higher = more relevant
```

BM25 finds "SKU-4471" via exact token match regardless of what it "means" — exactly the case dense
retrieval struggles with.

## Fusing Sparse and Dense: Reciprocal Rank Fusion

Once you have two independent rankings of the same candidate chunks — one from BM25, one from vector
similarity — you need to combine them into a single ranking. The scores aren't on comparable scales
(BM25 scores and cosine similarities live in different ranges and distributions), so naively summing
them is unreliable. **Reciprocal Rank Fusion (RRF)** sidesteps that by combining **ranks**, not raw
scores:

```
RRF_score(doc) = sum over each ranking r of  1 / (k + rank_r(doc))
```

where `rank_r(doc)` is the document's position (1st, 2nd, 3rd...) in ranking `r`, and `k` is a small
constant (commonly 60) that dampens the influence of very low ranks so a document ranked 1st isn't
overwhelmingly dominant versus one ranked 3rd.

```python
def reciprocal_rank_fusion(rankings: list[list[str]], k: int = 60) -> dict[str, float]:
    """rankings: list of ranked doc-id lists, one per retrieval method (best-first)."""
    scores: dict[str, float] = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return dict(sorted(scores.items(), key=lambda kv: kv[1], reverse=True))

fused = reciprocal_rank_fusion([dense_ranked_ids, bm25_ranked_ids])
```

A document that ranks well in **both** lists accumulates score from both terms and rises to the top
of the fused ranking; a document that only one method liked still gets some credit, which is exactly
the desired behavior — a chunk that's the *exact* SKU match (top of BM25, maybe mediocre in dense) and
a chunk that's the best *semantic* match (top of dense, maybe mediocre in BM25) should both surface
near the top, and RRF achieves that without needing to calibrate BM25 and cosine scores onto the same
scale.

## Applying This to the Virtual Liaison's Two RAG Pipelines

- **Cost-catalog RAG**: this is the strongest case for hybrid search on this platform. Clients ask
  price questions using a mix of natural language ("how much for a Japanese localization") and exact
  identifiers ("what's the price on SKU-4471"). Dense retrieval alone handles the first well and the
  second poorly; BM25 alone is the reverse. Hybrid + RRF handles both in one query path.
- **Project-data RAG**: project codenames ("Project Atlas") behave like SKUs — proper nouns an
  embedding model has no strong representation for. A client typing the exact codename should get an
  exact match; a client describing the project in general terms should still get a good semantic
  match. Hybrid search covers both without needing the system to first classify which kind of query
  it received.

```python
def hybrid_retrieve(query: str, top_k: int = 5):
    dense_ranked = [doc_id for doc_id, _ in dense_search(query, k=20)]
    bm25_ranked = [doc_id for doc_id, _ in bm25_search(query, k=20)]
    fused = reciprocal_rank_fusion([dense_ranked, bm25_ranked])
    return list(fused.keys())[:top_k]
```

Some vector databases (Pinecone included, via sparse-dense vectors on suitable index configurations)
support hybrid search natively; the fallback — and the version worth being able to explain and
implement from scratch, since it's the version an interviewer can ask you to whiteboard — is running
BM25 and dense retrieval as two independent passes and fusing the results in the application layer,
exactly as above.

## The Trade-off to Name Explicitly

Hybrid search isn't free: it means maintaining two retrieval indexes (a BM25/inverted index alongside
the vector index), running two searches per query instead of one, and an extra fusion step — added
latency and operational surface area. The right framing for an interview: hybrid search earns its
cost specifically when the query distribution includes a meaningful share of exact-identifier lookups
alongside natural-language questions, which is exactly the mix a cost-catalog and project-codename
system produces. For a purely conversational FAQ system with no structured identifiers in play, plain
dense retrieval is usually good enough, and hybrid search would be over-engineering.

## Tying It Back

When asked "hybrid search vs. pure dense retrieval — when does it matter," the answer is this
project's cost catalog: clients ask about SKUs and project codenames by exact identifier as often as
they describe them in natural language, and dense-only retrieval systematically underperforms on the
identifier case because embedding models don't have strong representations for arbitrary internal
codes. Combining BM25 (exact-match recall) with dense retrieval (semantic recall) via reciprocal rank
fusion covers both without requiring the system to first guess which kind of query it received.
`notebooks/03_hybrid_search_bm25_plus_dense.ipynb` builds this from scratch on a small synthetic
catalog and shows the fused ranking beating either method alone on queries that mix an exact SKU with
natural language.
