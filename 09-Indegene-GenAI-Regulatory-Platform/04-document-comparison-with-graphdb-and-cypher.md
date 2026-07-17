# 04 — Document Comparison with GraphDB & Cypher

## Why this chapter matters

"Country Specific Protocol Comparison" is the one module on this resume bullet that isn't primarily a
*generation* task — it's a *comparison* task, and the skills list calls out **GraphDB** and **Cypher
query language** specifically for it. That's a meaningful design signal: the candidate didn't reach
for a plain text diff. This chapter builds the case for why a graph representation fits this problem
better than a flat comparison, teaches graph-database and Cypher fundamentals from scratch, and works
through example queries that map directly onto "what changed for country X."

## The problem: comparing a global Protocol against a country variant

A global clinical trial Protocol defines the trial's design for every participating country. Individual
countries frequently need **local variations** on top of it — a stricter eligibility criterion required
by a national regulator, an additional consent disclosure mandated by local law, a different
permitted dosing range because of a local drug-label difference, an added local safety-monitoring
requirement. "Country Specific Protocol Comparison" is the task of surfacing exactly what's different
between the global Protocol and a given country's variant, so regulatory affairs and local site staff
know precisely what applies to them.

The naive approach — run a text diff between the global Protocol document and the country-variant
document — has real limitations for this content:

- **Text diffs are line/token-level, not requirement-level.** A single eligibility criterion might be
  reworded (different phrasing, same clinical meaning) without changing its substance, which a text
  diff flags as a change even though nothing regulatorily meaningful happened — or, conversely, a
  substantive change (a lab value threshold shifted) might be a small token-level edit that's easy to
  miss when scanning line-level diff noise.
- **Requirement dependencies are invisible in flat text.** Most Protocol sections reference other
  sections (an eligibility criterion that points to a specific lab test defined elsewhere, a dosing
  rule that depends on a body-weight category defined in a different table). A text diff has no
  concept of that relationship, so it can't tell you "this
  eligibility criterion changed, and it depends on a lab-reference-range requirement that *also*
  changed for this country" — which is exactly the kind of compound, easy-to-miss difference a
  reviewer most needs flagged.
- **Reordering false-positives.** Documents that reorganize section order between versions produce
  large text diffs that are mostly noise, obscuring the actual substantive differences.

## Why a graph fits better

The core insight: Protocol content isn't really a flat sequence of text — it's a network of
**requirements that reference and depend on each other**. Modeling it as a graph makes that structure
explicit and queryable:

- **Nodes** represent addressable units of the Protocol: sections, individual requirements
  (an inclusion criterion, a dosing rule, a safety-monitoring rule), and country-specific variant
  nodes attached to the requirements they modify.
- **Edges** represent relationships: `HAS_SECTION`, `HAS_REQUIREMENT`, `REFERENCES` (one requirement
  depending on another), and — critically for this task — `MODIFIES` or `OVERRIDES` edges connecting
  a country-variant node to the global requirement node it changes.

With that model, "what's different for Country X" stops being a fuzzy text-comparison problem and
becomes a precise graph traversal: *find all requirement nodes that have an incoming `MODIFIES`/
`OVERRIDES`/`ADDS` edge from a Country X variant node, and follow `REFERENCES` edges outward to
surface anything downstream that's affected too.* That last part — traversing to dependent
requirements — is the capability a flat text diff fundamentally cannot offer, and it's the strongest,
most concrete reason to reach for a graph representation in an interview answer.

## Graph database fundamentals

A **graph database** (Neo4j is the most common example, and a reasonable one to name if asked) stores
data natively as nodes, relationships, and properties on both, rather than as rows in tables joined at
query time. In production this ran as a self-hosted graph database service, kept isolated per client
(Eli Lilly's protocol graph and AstraZeneca's protocol graph never share an instance or namespace) for
the same confidentiality reasons that drove per-client S3 buckets and IAM roles elsewhere in the
platform. The practical benefit for this use case: relationship traversal (multi-hop, e.g. "find
everything two hops away from this requirement") is a first-class, efficient operation in a graph
database, where in a relational database the equivalent would be a chain of joins that gets slower
and harder to write correctly as the traversal depth grows.

A minimal schema for this problem:

```
(:Section {name, number})
(:Requirement {id, text, type})   // type: "eligibility" | "dosing" | "safety" | ...
(:CountryVariant {country, id})

(:Section)-[:HAS_REQUIREMENT]->(:Requirement)
(:Requirement)-[:REFERENCES]->(:Requirement)
(:CountryVariant)-[:MODIFIES]->(:Requirement)
(:CountryVariant)-[:ADDS]->(:Requirement)
```

## Cypher: the query language

**Cypher** is Neo4j's declarative graph query language, built around pattern matching: you describe a
shape of nodes and relationships you're looking for using an ASCII-art-like syntax, and Cypher returns
everything in the graph that matches that shape. The core clauses:

- **`MATCH`** — describe the pattern to find, e.g. `(a)-[:REL]->(b)`.
- **`WHERE`** — filter matched patterns by property values.
- **`RETURN`** — specify what to return from the matches (whole nodes, specific properties,
  aggregates).

### Example 1: find every requirement modified or added for a given country

```cypher
MATCH (c:CountryVariant {country: "Germany"})-[r:MODIFIES|ADDS]->(req:Requirement)
RETURN req.id, req.type, req.text, type(r) AS change_type
```

This is the graph-native equivalent of "run a diff between the global Protocol and the Germany
variant" — except it returns exactly the requirement-level differences, tagged by whether they're a
modification or a wholly new addition, with no line-level noise from reordering or rewording.

### Example 2: find requirements that depend on something changed for a country (the diff a text tool cannot do)

```cypher
MATCH (c:CountryVariant {country: "Germany"})-[:MODIFIES]->(changed:Requirement)
MATCH (dependent:Requirement)-[:REFERENCES]->(changed)
RETURN dependent.id, dependent.text, changed.id AS depends_on_changed_requirement
```

This surfaces the compound case flagged above: any requirement that references a requirement that
was itself changed for this country — the kind of ripple effect a reviewer is likely to miss reading
documents linearly, and exactly the reason a flat diff isn't sufficient for this task.

### Example 3: compare two countries against each other

```cypher
MATCH (c1:CountryVariant {country: "Germany"})-[:MODIFIES|ADDS]->(req:Requirement)
MATCH (c2:CountryVariant {country: "Japan"})-[:MODIFIES|ADDS]->(req)
RETURN req.id, req.text
```

Because both countries' variant relationships point at the *same underlying requirement nodes*,
finding requirements that both countries modify (or diverge on) is a straightforward pattern match —
this generalizes the comparison from "global vs. one country" to "country vs. country" almost for
free, which is a natural extension question an interviewer might probe.

### Example 4: count changes by type, across all countries

```cypher
MATCH (c:CountryVariant)-[r:MODIFIES|ADDS]->(req:Requirement)
RETURN c.country, req.type, count(*) AS change_count
ORDER BY c.country, change_count DESC
```

Useful as a reviewer-facing summary: "Germany has 4 eligibility-criteria changes and 2 dosing
changes" is a much faster starting point for a regulatory reviewer than reading every requirement.

## From Python graph-diff to Cypher: the same logic, two representations

The notebook `03_graphdb_cypher_queries_demo.ipynb` builds a small in-memory graph using plain Python
dictionaries (adjacency lists) representing a global protocol and a country variant, implements a
graph-diff function in pure Python that walks that structure to find additions/removals/modified
requirements, and pairs each Python function with the equivalent Cypher query shown above in a
markdown cell — so you can see the same logic expressed both ways. In an interview, being able to say
"here's the traversal in plain Python, and here's the same pattern in Cypher against a real Neo4j
instance" demonstrates that the value is in the *graph modeling*, not merely in knowing Neo4j's
syntax.

## Tying it back

"Why model this as a graph instead of a simple diff" is a question you should expect directly, and
the answer has three layers: (1) Protocol requirements reference each other, and a graph is the
natural structure for that, letting you traverse from a change to its downstream effects; (2)
country-specific variants naturally attach as edges (`MODIFIES`/`ADDS`) onto shared requirement nodes,
which scales cleanly to comparing many countries against the same global Protocol or against each
other, unlike pairwise document diffing; (3) Cypher gives regulatory reviewers and engineers a
precise, declarative way to ask "what changed for this country, and what does that affect" — a
question a line-level text diff structurally cannot answer.
