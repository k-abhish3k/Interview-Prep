# 02 — LangChain Core Abstractions & LCEL

## Why this chapter matters

The resume lists LangChain, LCEL, and LangServe as core skills, and "orchestrating calls to Azure
OpenAI" is exactly what LangChain is for. Interviewers who've used LangChain will often probe with
"why LangChain instead of just calling the OpenAI SDK directly?" — you need a real answer, not just
"it's the popular library." This chapter gives you that answer, plus enough LCEL fluency to write
small pipelines live in an interview if asked.

## Why Use a Framework At All

Calling `openai.ChatCompletion.create(...)` directly works for a single, simple call. A production
chatbot needs: prompt templating with variable substitution, swappable model backends (so you can
point at Azure OpenAI in prod and a local/fake model in tests), output parsing into structured
objects, conversation memory, retrieval integration, streaming, batching, and retries/observability —
all composed together and reusable across the codebase. LangChain provides standard abstractions for
each of these pieces so you're not hand-rolling glue code, and it has first-class support for Azure
OpenAI specifically (`AzureChatOpenAI`), which matters because Azure OpenAI's client setup (endpoint,
deployment name, API version) differs from public OpenAI's.

## Core Abstractions

### PromptTemplate / ChatPromptTemplate

A `PromptTemplate` separates the *shape* of a prompt from the *data* filled into it — critical once
you have dozens of prompts across a codebase and need to version and test them independently of the
code that calls them.

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an internal assistant for {client_name}. "
               "Only answer using the provided context. If you don't know, say so."),
    ("human", "Context:\n{context}\n\nQuestion: {question}"),
])

# .format_messages(...) or, more commonly, passed straight into an LCEL chain
messages = prompt.format_messages(
    client_name="Acme Bank",
    context="Refunds are processed within 5-7 business days.",
    question="How long do refunds take?",
)
```

This is where the system/user prompt design from Chapter 1 actually gets implemented in code, with
`{placeholders}` for the retrieved context and the user's question.

### LLM / ChatModel Wrappers

LangChain wraps every provider behind a common interface (`invoke`, `stream`, `batch`), so swapping
providers is a one-line change:

```python
from langchain_openai import AzureChatOpenAI

llm = AzureChatOpenAI(
    azure_deployment="gpt-4o-chatbot-prod",   # the Azure OpenAI *deployment* name, not the model name
    api_version="2024-05-01-preview",
    temperature=0.2,
)
```

The key Azure-specific detail worth knowing cold: in Azure OpenAI you call a **deployment**, not a
raw model name — your organization deploys a specific model version under a deployment name inside
its Azure OpenAI resource, and the client SDK targets that deployment. This is a common interview
gotcha ("what's the difference between calling OpenAI directly vs Azure OpenAI in code?").

### Output Parsers

Raw LLM output is a string. Output parsers turn that string into a typed Python object your app can
use — critical for the "structured output prompting" pattern from Chapter 1.

```python
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from pydantic import BaseModel

class Answer(BaseModel):
    answer: str
    source: str
    confidence: float

parser = JsonOutputParser(pydantic_object=Answer)
```

`StrOutputParser` is the simplest case — it just extracts the plain text of the model's response,
which is what you use when you don't need structure (e.g., a conversational reply the UI just
renders as text).

### Memory

Memory objects manage the conversation history that gets re-injected into the prompt each turn
(recall from Chapter 1: the model has no memory of its own). Older LangChain versions had dedicated
`ConversationBufferMemory` / `ConversationSummaryMemory` classes; current LangChain favors
explicitly managing a `ChatMessageHistory` and wiring it in with
`RunnableWithMessageHistory`, which is more transparent about exactly what's being sent to the model
each call — important when you're trying to control token cost and context window usage (see
Chapter 3, and the from-scratch memory implementation in
`notebooks/03_simple_chatbot_with_memory.ipynb`).

## LCEL — LangChain Expression Language

LCEL is the modern way to compose these pieces using the `|` (pipe) operator, borrowed conceptually
from Unix pipes: the output of one step becomes the input of the next.

```python
chain = prompt | llm | StrOutputParser()

response = chain.invoke({
    "client_name": "Acme Bank",
    "context": "Refunds are processed within 5-7 business days.",
    "question": "How long do refunds take?",
})
```

Every piece in that pipe — the prompt template, the chat model, the parser — implements the same
`Runnable` interface, which is the whole trick that makes `|` work: `Runnable` defines `invoke`,
`batch`, `stream`, and their async equivalents (`ainvoke`, `abatch`, `astream`), and `|` is just
syntactic sugar for composing two `Runnable`s into a `RunnableSequence` that feeds one's output into
the other's input.

### Why LCEL matters over "legacy" chains

Before LCEL, LangChain had purpose-built `Chain` classes (`LLMChain`, `SequentialChain`, etc.), each
with its own bespoke API. LCEL replaced that with one uniform interface everything implements, which
gets you, for free, without extra code:

- **Streaming**: `chain.stream(...)` yields output incrementally, token by token, as soon as the
  underlying model produces it — essential for a chatbot UI, since users perceive a response that
  starts appearing in 300ms as far faster than one that appears all-at-once after 4 seconds, even if
  total generation time is identical.

  ```python
  for chunk in chain.stream({"client_name": "Acme Bank", "context": "...", "question": "..."}):
      print(chunk, end="", flush=True)
  ```

- **Batching**: `chain.batch([...])` runs multiple inputs efficiently (with automatic concurrency),
  useful for offline evaluation runs or bulk-processing a queue of questions.
- **Async support** for high-throughput backend services handling many concurrent users.
- **Composability**: since everything is a `Runnable`, you can nest chains inside chains, branch with
  `RunnableBranch`, run steps in parallel with `RunnableParallel` (e.g., running retrieval and a
  query-classification step concurrently), and attach retry/fallback behavior declaratively.
- **LangServe compatibility**: chains built with LCEL can be deployed directly as a REST API via
  LangServe with minimal glue code, since LangServe expects `Runnable`-shaped objects — directly
  relevant if the Capco chatbot's backend exposed its chain as an HTTP endpoint for the
  Streamlit/Chainlit frontend to call.

### A Slightly Fuller Chain

Putting it together the way a RAG-lite version of the Capco chatbot might look, without the actual
retrieval step (covered in Chapter 3):

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import AzureChatOpenAI

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an internal assistant for {client_name}. "
               "Only answer using the provided context. If you don't know, say so."),
    ("human", "Context:\n{context}\n\nQuestion: {question}"),
])
llm = AzureChatOpenAI(azure_deployment="gpt-4o-chatbot-prod", temperature=0.2)
chain = prompt | llm | StrOutputParser()

# streaming into a UI
for token in chain.stream({
    "client_name": "Acme Bank",
    "context": retrieved_context,   # from Azure AI Search, see Chapter 3
    "question": user_message,
}):
    yield token
```

## Tying It Back

When asked "how did you structure the chatbot's code," this is the answer: a `ChatPromptTemplate`
encoding the system/user prompt design, an `AzureChatOpenAI` wrapper pointed at the client's Azure
OpenAI deployment, an output parser matching whatever the UI needed to render, all composed with `|`
into an LCEL chain that streamed tokens back to the Streamlit/Chainlit frontend for responsive UX.
The Text-to-Math agent in Chapter 5 builds on these exact same `Runnable` primitives, just with a
tool-calling loop wrapped around the chain instead of a single straight-through pipe.
