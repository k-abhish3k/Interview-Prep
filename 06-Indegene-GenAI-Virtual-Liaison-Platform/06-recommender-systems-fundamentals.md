# 06 — Recommender Systems Fundamentals

## Why this chapter matters

"Multiple Recommender Systems" is the last named facility on the resume bullet, and it's easy to
under-prepare for because recommenders feel like a different discipline from RAG/LLM work. In this
platform they're not separate — the same embeddings you're already computing for RAG (Chapter 1) do
double duty as recommendation features. This chapter covers the three standard recommender paradigms,
maps each onto a plausible use inside the Virtual Liaison ("similar past projects," "relevant catalog
items"), and explains why an embedding-based, content-based approach is the practical default here
even though collaborative filtering is worth knowing about and naming as a limitation.

## Three Paradigms

### 1. Content-Based Filtering

Recommend items similar to a given item (or similar to a profile built from items the user has
engaged with), based on the items' own **features/content** — not on what other users did.

For "recommend past projects similar to this new request," content-based filtering is a direct fit:
represent each project as a feature vector (asset type, languages, region, description text via
TF-IDF or embeddings) and recommend the nearest past projects to the new request's vector by cosine
similarity.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

project_profiles = [
    "Promotional deck localization Japanese Korean APAC region pharma launch",
    "Regulatory submission package French German EU compliance review",
    "Training deck localization Japanese APAC oncology launch",
    # ... one text profile per past project
]
vectorizer = TfidfVectorizer(stop_words="english")
profile_matrix = vectorizer.fit_transform(project_profiles)

new_request = "Localize promotional materials into Japanese for APAC oncology product"
new_vector = vectorizer.transform([new_request])

similarities = cosine_similarity(new_vector, profile_matrix).ravel()
top_n_idx = similarities.argsort()[::-1][:3]   # top 3 most similar past projects
```

Content-based recommenders work well from day one — a brand-new project with zero interaction history
can still be recommended and can still generate recommendations, because everything is driven by the
item's own content, not by accumulated usage data. That "cold start" resilience is exactly why it's
the right default for this platform: the recommender needs to work the first time a client uses the
assistant, not after months of accumulated interaction logs.

### 2. Collaborative Filtering

Recommend items based on patterns across **many users' behavior**, not item content — "clients who
requested similar past projects also requested catalog item Y." Two classic forms:

- **User-based**: find clients with similar request histories to the current client, recommend what
  those similar clients requested.
- **Item-based**: find catalog items frequently requested *together* with items the current client has
  already engaged with, independent of content similarity — matrix factorization (e.g., learning
  latent user and item vectors such that their dot product approximates observed interaction
  strength) is the standard modern technique here.

Collaborative filtering's strength is that it can surface non-obvious associations content-based
filtering would never find (two catalog items that are frequently bundled for business reasons that
have nothing to do with textual similarity), but it has a hard structural weakness: the **cold-start
problem** — it needs a substantial history of interactions to find meaningful patterns, and produces
poor or no recommendations for new clients, new projects, or a platform still early in adoption. For
a platform serving specific clients like Eli Lilly and AstraZeneca, that interaction history is also
client-scoped by design (Chapter 2's namespace isolation means one client's interaction data can't be
borrowed to help another client's cold start) and may simply be too sparse per client to make
collaborative filtering reliable on its own — worth naming explicitly as the reason it's a secondary,
not primary, technique here.

### 3. Embedding-Based / LLM-Embedding Similarity Recommenders

A middle ground that generalizes content-based filtering using the same dense embeddings already
being computed for RAG (Chapter 1): represent projects, cost-catalog items, or client requests as
embedding vectors from the same (or a compatible) embedding model used for retrieval, and recommend
by nearest-neighbor search in that embedding space — literally the same ANN search from Chapter 2,
repurposed.

```python
# Reusing the same embedding model and Pinecone index style as the RAG pipeline (Chapter 2)
new_request_embedding = embedder.embed_query(new_request_text)

similar_projects = index.query(
    vector=new_request_embedding,
    top_k=3,
    namespace=client_id,
    filter={"doc_type": {"$eq": "project_profile"}},
    include_metadata=True,
)
```

This is the most natural fit for a platform that already has an embedding + vector-store
infrastructure built for RAG: no separate recommendation-specific model or pipeline is needed, and
embeddings capture semantic similarity (a "promotional deck for an oncology launch" and a "marketing
deck for a cancer therapy launch" land close together) that pure TF-IDF content-based filtering, which
only matches on lexical overlap, would miss. It inherits content-based filtering's cold-start
resilience (a similarity search needs no interaction history, only the item's own text/embedding)
while being more semantically robust than raw TF-IDF.

## Choosing an Approach for This Platform

| | Content-based (TF-IDF) | Collaborative filtering | Embedding similarity |
|---|---|---|---|
| Works with no interaction history | Yes | No (cold start) | Yes |
| Captures semantic (not just lexical) similarity | No | N/A | Yes |
| Needs a large user base to work well | No | Yes | No |
| Reuses existing RAG infrastructure | Partially | No | Yes — same embeddings/index |
| Finds non-obvious cross-item associations | No | Yes | Partially (only via semantic proximity) |

The practical design for "multiple Recommender Systems" on this platform: **embedding-similarity
recommenders as the primary mechanism** for both "similar past projects" and "relevant catalog items"
(reusing the RAG embedding + Pinecone infrastructure directly), with **collaborative-filtering signals
layered in opportunistically** once enough interaction history accumulates — e.g., boosting catalog
items that are frequently requested alongside items in the current recommendation set, as a secondary
re-ranking signal rather than the primary retrieval mechanism. This mirrors the RAG/hybrid-search
lesson from Chapter 3: don't rely on one technique when two complementary ones, fused, cover more
ground.

## Evaluating a Recommender With Sparse Historical Data

This is a common interview follow-up: "how do you evaluate a recommender system when you don't have
much interaction data yet?" Standard recommender metrics — precision@K, recall@K, NDCG — all assume
you have a reasonably large held-out set of "ground truth" interactions to check recommendations
against, which a young, sparse-data platform doesn't have. Practical alternatives:

- **Proxy relevance labels**: use content/business-rule similarity (same asset type, same client
  industry vertical, overlapping languages) as a weak ground truth to compute precision@K against,
  acknowledging it's an approximation, not true user preference.
- **Human-in-the-loop evaluation**: have project managers or liaisons rate a sample of recommendations
  for relevance — expensive per-sample but far more trustworthy than a proxy metric when data is
  sparse.
- **Online metrics over offline metrics**: track whether recommended items actually get selected/
  accepted in production (click-through/accept rate on "similar past projects" suggestions) rather
  than trying to validate purely offline — this also naturally accumulates the interaction data needed
  to eventually make collaborative filtering viable.
- **A/B or shadow testing**: show recommendations without acting on them (or to a subset of sessions)
  and measure whether they correlate with what the client/liaison chose anyway, before fully relying
  on them.

The honest interview answer is that with sparse data, you evaluate a recommender the way you'd
evaluate any system without ground truth: proxy metrics plus human judgment first, then transition to
real online metrics as usage accumulates — and you say this explicitly rather than pretending
precision@K was available from day one.

## Tying It Back

When asked "how did the recommender systems work," the answer is: embedding-based similarity search
reusing the same embedding model and Pinecone infrastructure already built for RAG (Chapter 1/2) — no
separate recommendation pipeline — recommending similar past projects and relevant cost-catalog items
by nearest-neighbor search over project/catalog embeddings, chosen specifically because it works from
day one without requiring an interaction history, unlike collaborative filtering, which was
considered but set aside (or layered in only as a secondary signal) due to the cold-start problem on a
platform without years of accumulated per-client usage data. `notebooks/06_simple_recommender_system.
ipynb` implements the content-based version from scratch with TF-IDF and cosine similarity over
synthetic project profiles.
