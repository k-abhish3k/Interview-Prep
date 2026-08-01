# 02 — Vector Databases & Pinecone

## Why this chapter matters

"RAG-enabled Project-based data retrieval using langchain & pinecone" names Pinecone specifically —
expect an interviewer to ask "why Pinecone and not [pgvector / FAISS / Elasticsearch]" and "how did
you make sure Client A never saw Client B's data." Both questions are really about the same
underlying concept: how a vector database indexes for fast approximate search, and how it partitions
data for multi-tenancy. This chapter builds both from first principles, then maps them onto Pinecone's
concrete API.

## Why a Specialized Vector Database

Chapter 1 established that RAG needs to find the nearest vectors to a query vector, fast, at scale.
A relational database can store vectors as arrays, but a naive nearest-neighbor search there is
**brute force**: compare the query against every single row. That's O(n) per query — fine for a few
thousand chunks, unworkable once a platform indexes project histories and a cost catalog across many
clients, potentially millions of chunks. Vector databases exist to make that search **sub-linear**
using approximate nearest neighbor (ANN) algorithms, while also handling the surrounding operational
concerns — metadata filtering, namespacing, upserts, horizontal scaling — that a plain ANN library
like FAISS doesn't provide out of the box (FAISS is an indexing library, not a managed database: no
built-in persistence, filtering, or multi-tenant isolation).

## ANN Search and HNSW, Conceptually

Exact nearest-neighbor search guarantees the true closest vectors but scales linearly with data size.
**Approximate** nearest-neighbor search trades a small amount of accuracy for a large speedup by
searching a cleverly constructed index instead of every vector.

**HNSW (Hierarchical Navigable Small World)** is the dominant ANN algorithm behind most modern vector
databases, including Pinecone's pod-based indexes. The intuition:

- Vectors are organized into a multi-layer graph. The top layer has very few nodes with long-range
  connections (like an "express lane"); each layer below has progressively more nodes and
  shorter-range connections, down to the bottom layer, which contains every vector.
- A search starts at the top layer, greedily walks toward the query vector using the sparse long-range
  edges to cover a lot of "distance" cheaply, then drops down a layer and repeats with progressively
  finer-grained edges — conceptually similar to how a highway system gets you close to a destination
  fast, then local roads handle the final approach.
- This gets query time down to roughly logarithmic in the number of vectors, instead of linear —
  the difference between a retrieval call taking milliseconds versus seconds once the index holds
  millions of chunks.

The trade-off to name explicitly in an interview: HNSW (and ANN in general) is **approximate** — it
can occasionally miss the true single-best match in exchange for large speed gains, and index build
time/memory grows with graph density. That's an acceptable trade for "top-5 relevant project status
chunks," since a slightly-less-than-perfect top-5 rarely changes the generated answer, but would be a
bad trade for something requiring exact retrieval guarantees.

## Pinecone as the Worked Example

Pinecone is a managed vector database — it owns the ANN indexing, sharding, and replication so the
platform team doesn't run that infrastructure itself, which is a reasonable choice for a project team
optimizing for delivery speed over infrastructure ownership.

### Index Creation

An index is the top-level container — you choose a dimensionality matching your embedding model's
output size and a similarity metric (cosine, dot product, or Euclidean).

```python
from pinecone import Pinecone, ServerlessSpec

pc = Pinecone(api_key="...")

pc.create_index(
    name="indegene-project-data",
    dimension=1536,              # must match the embedding model's output dimension
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
)
index = pc.Index("indegene-project-data")
```

Getting `dimension` wrong is a classic gotcha: it must exactly match whatever embedding model
produced the vectors (e.g., 1536 for `text-embedding-3-small`), and it can't be changed after index
creation without rebuilding the index — worth remembering if asked about migration cost when
switching embedding models.

### Upsert

Chunks are embedded (Chapter 1), then written to the index along with an ID and **metadata** — the
fields you'll want to filter on later.

```python
index.upsert(
    vectors=[
        {
            "id": "proj-atlas-status-0042",
            "values": embedding_vector,          # e.g. 1536-dim float list
            "metadata": {
                "client_id": "acme-pharma",
                "project_id": "atlas",
                "doc_type": "status_update",
                "date": "2026-07-10",
            },
        },
        # ... more chunks
    ],
    namespace="acme-pharma",
)
```

"Upsert" (update-or-insert) means re-writing the same ID overwrites the existing vector and metadata —
the mechanism that keeps project status current: when a project moves to a new stage, the chunk
representing "current status" gets re-embedded and upserted under the same ID rather than growing an
ever-longer history that has to be re-summarized on every read.

### Query with Filters

A query embeds the user's question, then asks Pinecone for the top-K nearest vectors, optionally
restricted by a metadata filter.

```python
results = index.query(
    vector=query_embedding,
    top_k=5,
    namespace="acme-pharma",
    filter={
        "project_id": {"$eq": "atlas"},
        "doc_type": {"$in": ["status_update", "revision_note"]},
    },
    include_metadata=True,
)
```

Metadata filtering is applied **before or alongside** the ANN search (not as a slow post-filter over
already-ranked results), which is what makes it usable for hard isolation requirements rather than
just a relevance nicety.

## Multi-Tenancy: Isolating Client and Project Data

This is the part of the Pinecone chapter interviewers care about most for a platform like this one,
because a Virtual Liaison that leaks one client's project data into another client's answer is a
trust-ending failure, not a minor bug. Two mechanisms, used together:

- **Namespaces** partition an index into fully separate sub-indexes that share the same underlying
  infrastructure but are queried independently — a query against namespace `acme-pharma` physically
  cannot return a vector stored under namespace `globex-bio`. This is the strongest isolation
  Pinecone offers short of separate indexes entirely, and it maps naturally onto "one namespace per
  client," since client boundaries are the hardest isolation requirement (project boundaries within a
  client are a softer, business-logic-level separation).
- **Metadata filters** (`project_id`, `doc_type`, `region`, etc.) then narrow *within* a client's
  namespace to the specific project or document type relevant to the current query — this is a
  relevance filter as much as an isolation one, since two projects for the same client shouldn't
  bleed into each other's status answers either.

A reasonable design, and the one worth describing in an interview: **namespace per client**, plus
**metadata filter per project** within that namespace. That keeps the hard security boundary
(different companies' data) enforced at the infrastructure level, where it's easy to audit and hard
to get wrong in application code, while keeping the softer boundary (different projects for the same
client) flexible enough to support cross-project queries when a client legitimately asks something
like "summarize all my active projects this quarter."

```python
# Retrieval always scoped to the requesting client's namespace — never client-selectable
def retrieve_for_client(query_embedding, client_id, project_id=None, top_k=5):
    filt = {"project_id": {"$eq": project_id}} if project_id else {}
    return index.query(
        vector=query_embedding,
        top_k=top_k,
        namespace=client_id,     # hard isolation boundary
        filter=filt,             # soft, query-specific narrowing
        include_metadata=True,
    )
```

The `namespace` argument should be derived from the authenticated session/client context server-side
— never taken from user input — which is the kind of detail that separates a "yes I used Pinecone"
answer from one that demonstrates you understood the security model.

### Making This Concrete: Eli Lilly and AstraZeneca

This isn't a hypothetical multi-tenancy exercise — the Virtual Liaison platform served **Eli Lilly**
and **AstraZeneca** in production on the same Pinecone index and the same orchestration codebase, and
those two companies are competing pharma companies. In practice that meant a namespace per client
(`eli-lilly`, `astrazeneca`, in place of the generic `acme-pharma` used elsewhere in this chapter),
with the namespace resolved server-side from the authenticated session on every single retrieval call
— project RAG, cost-catalog RAG, and the embedding-similarity recommenders in Chapter 6 all reuse the
same namespace-resolution logic, so there's one code path to audit, not five.

The stakes of getting this wrong are worth stating plainly: a cross-namespace query — Eli Lilly's
session somehow retrieving an AstraZeneca project note or cost-catalog line — is not a bug you file
and fix next sprint. It's a **client-trust-ending incident**: two competing pharma companies' project
data, pricing, and unpublished plans surfacing in front of the wrong company's team is the kind of
failure that ends an engagement, not just a support ticket. That's why the namespace boundary is
enforced at the infrastructure level (Pinecone physically cannot return another namespace's vectors)
rather than only as an application-layer `WHERE client_id = ...` filter that a coding mistake could
bypass — defense should not rely solely on every engineer remembering to add the filter correctly on
every new query path.

## Tying It Back

When asked "how did you keep Pinecone data isolated between clients," the answer is: one namespace
per client as the hard boundary, metadata filters (`project_id`, `doc_type`) for finer-grained
relevance within that namespace, and the namespace value always derived server-side from the
authenticated request context. `notebooks/02_pinecone_vector_search_demo.ipynb` implements a
`FakePineconeIndex` class matching this same `create_index` / `upsert` / `query` shape, entirely
in-memory, so you can see the namespace + filter logic run end to end without a real API key.
