# 04 — Summarization Techniques

## Why this chapter matters

"Concise project tracking & summarization" is its own named facility on the resume, separate from
retrieval. A client asking "give me a quick update on Project Atlas" doesn't want the five most
similar raw status chunks dumped back at them (that's what RAG retrieval alone gives you) — they want
those chunks **synthesized** into a short, coherent narrative. This chapter covers the summarization
techniques that make that possible, and — critically — the failure mode that shows up once a
project's history grows past what fits in a single LLM context window, which is the part interviewers
actually want to hear you reason through.

## Extractive vs. Abstractive Summarization

- **Extractive summarization** selects and stitches together existing sentences/spans from the source
  text, verbatim, choosing the ones judged most representative or important. It never generates new
  wording, which makes it fast, cheap, and impossible to hallucinate — but the output can read
  disjointed, since sentences pulled from different points in a project's history don't always flow
  as a narrative.
- **Abstractive summarization** generates new text that paraphrases and condenses the source, the way
  a person would summarize a meeting to a colleague who wasn't there. This is what an LLM does by
  default when asked to summarize, and it reads far more naturally — "Project Atlas is on track for a
  Japan launch in Q3; French localization is delayed two weeks pending legal review" is a sentence no
  extractive method would produce verbatim from the source notes, because it's a synthesis across
  several of them.

For "give me a concise update on this project," abstractive summarization via the LLM is the right
default — clients want a synthesized narrative, not a bag of quoted sentences. Extractive methods are
still useful as a **pre-filtering step**: cheaply score and select the most important sentences/notes
*before* handing them to the LLM, which reduces token cost and keeps the abstractive step focused on
signal rather than noise. `notebooks/04_summarization_demo.ipynb` implements exactly this kind of
extractive scorer from scratch, using TF-IDF term weight as a proxy for sentence importance.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
import numpy as np

def extractive_top_sentences(sentences: list[str], top_n: int = 5) -> list[str]:
    vectorizer = TfidfVectorizer(stop_words="english")
    tfidf = vectorizer.fit_transform(sentences)
    scores = np.asarray(tfidf.sum(axis=1)).ravel()   # simple importance proxy
    top_idx = np.argsort(scores)[::-1][:top_n]
    return [sentences[i] for i in sorted(top_idx)]    # keep original order
```

## The Real Problem: Project Histories Longer Than the Context Window

A project that's been running for months accumulates a long history — dozens of status updates,
revision notes, and client exchanges. Stuffing the entire history into a single prompt eventually
exceeds the LLM's context window, and even before hitting that hard limit, very long contexts degrade
generation quality (models attend less reliably to information buried in the middle of a very long
prompt — the well-documented "lost in the middle" effect). Summarizing a long project history isn't a
single LLM call; it's a **chain** of calls with an explicit control flow. Two standard patterns:

### Map-Reduce Summarization

1. **Map**: split the project history into chunks (Chapter 1's chunking strategies apply directly —
   one chunk per status update or a fixed token size), and summarize each chunk **independently and
   in parallel**.
2. **Reduce**: combine the per-chunk summaries into a single final summary — either in one more LLM
   call (if the combined summaries fit in the context window) or recursively, reducing in stages, if
   they don't.

```python
def map_reduce_summarize(chunks: list[str], summarize_fn, combine_fn) -> str:
    # Map: summarize each chunk independently (can run concurrently)
    partial_summaries = [summarize_fn(chunk) for chunk in chunks]
    # Reduce: combine partial summaries into one
    return combine_fn(partial_summaries)
```

Map-reduce is **embarrassingly parallel** at the map step — every chunk's summary is independent of
every other, so all the map calls can run concurrently, which matters for latency when a project
history has many chunks. Its weakness: because each chunk is summarized in isolation, information that
only makes sense in relation to an earlier or later chunk (e.g., "the delay mentioned here was later
resolved three updates on") can get lost or fragmented across independent summaries, and only gets
reconciled — if at all — at the reduce step.

### Refine (Sequential) Summarization

1. Summarize the first chunk.
2. For each subsequent chunk, ask the LLM to **refine** the running summary given the new chunk's
   content — "here is the summary so far; here is the next update; produce an updated summary."
3. Repeat until every chunk has been folded in.

```python
def refine_summarize(chunks: list[str], refine_fn) -> str:
    summary = refine_fn(previous_summary="", new_chunk=chunks[0])
    for chunk in chunks[1:]:
        summary = refine_fn(previous_summary=summary, new_chunk=chunk)
    return summary
```

Refine preserves narrative continuity better than map-reduce, since each step has the full running
context — the "delay resolved three updates later" example above naturally gets reconciled because
the model sees both the original delay note and its resolution in sequence. The cost is that refine is
**strictly sequential** (each step depends on the previous summary), so it can't be parallelized and
scales linearly in latency with the number of chunks — a real concern for a very long project history
answered synchronously in a chat interface.

### Choosing Between Them for Project Tracking

The practical framing for an interview: **map-reduce when speed matters and chunks are largely
independent** (e.g., summarizing five unrelated projects in parallel for a portfolio overview), and
**refine when narrative coherence matters more than latency** (e.g., "walk me through how Project
Atlas got to its current status" — a question where the *evolution* of the project, not just its
current snapshot, is the point). In practice, a reasonable production design caches and incrementally
updates the refined summary as new status updates arrive, rather than re-summarizing full history on
every request — turning what would be an O(project history length) operation per query into an O(1)
lookup plus a cheap incremental refine step whenever something actually changes.

```python
from langchain_core.prompts import ChatPromptTemplate

refine_prompt = ChatPromptTemplate.from_messages([
    ("system", "You maintain a running summary of a client project for internal tracking."),
    ("human",
     "Current summary:\n{previous_summary}\n\n"
     "New update:\n{new_chunk}\n\n"
     "Produce an updated concise summary incorporating the new update."),
])
```

## Tying It Back

When asked "how did you implement project summarization," the answer distinguishes the two
techniques and explains the choice: abstractive summarization (via the LLM) for a natural, readable
update rather than extractive sentence-pulling, with an extractive TF-IDF pre-filter to control token
cost on very long histories, and a refine-style chain — rather than plain map-reduce — chosen because
project status is inherently sequential (each update builds on the last), so preserving that narrative
continuity mattered more than the parallelism map-reduce would have offered. In production, this
summary would be maintained incrementally rather than recomputed from scratch on every "give me an
update" request. `notebooks/04_summarization_demo.ipynb` implements both the extractive scorer and a
mocked map-reduce chain over a synthetic multi-paragraph project history, fully offline.
