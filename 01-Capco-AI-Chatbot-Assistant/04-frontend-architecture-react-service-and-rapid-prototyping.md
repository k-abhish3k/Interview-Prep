# 04 — Frontend Architecture: `react-service` and Rapid Prototyping

## Why this chapter matters

This chapter exists because of a real interview correction. An interviewer asked a natural follow-up
about the chatbot's user interface, and the honest answer is not "Streamlit or Chainlit" — it's a
**dedicated frontend microservice called `react-service`**: a React single-page application, built and
deployed as its own service, talking to the FastAPI backend over an HTTP/streaming API. That's a
**confirmed fact about the real production platform**, not an inference — state it plainly if asked
"what did the actual UI look like in production." Chainlit and Streamlit are still genuine, resume-worthy
skills — they're exactly what a backend-focused developer reaches for during a rapid internal-tool or
proof-of-concept phase — but they were not what shipped to HSBC as the production chatbot interface. This
chapter covers both halves honestly: what Chainlit/Streamlit are for (Part 1) and what the real
production frontend actually is (Part 2).

> **A note on confidence levels in this chapter.** That the production UI is a separate `react-service`
> microservice — not Chainlit/Streamlit — is confirmed. The specific implementation details in Part 2
> (the exact SSE event-shape JSON, the specific MSAL.js scope string, the specific CORS configuration) are
> a **typical/recommended architecture** for this class of separation, written to be technically precise
> and defensible in an interview, not a verified line-by-line description of `react-service`'s actual
> source code. Treat the *shape* of the design as solid; treat exact field names/values the way you would
> any other "illustrative" detail elsewhere in this course.

## Part 1 — Chainlit and Streamlit: what they're actually for

Both Streamlit and Chainlit are genuinely on the resume, and both are exactly the tools a
backend-focused developer reaches for during a pilot/POC phase, an internal demo, or a personal
project — before a client (or a consultancy's own delivery plan) commits to a dedicated frontend
codebase and a frontend team. That's the honest scope of these two tools: **rapid internal-tool and
prototyping skillset**, not the production HSBC chatbot UI. It's a genuinely useful thing to be able to
say plainly in an interview: *"Chainlit and Streamlit are what I'd reach for to get a working chat
experience in front of a stakeholder in hours, entirely in Python, before any frontend investment is
justified — the actual production chatbot at HSBC shipped through a separate React frontend service,
which is a different point in the same tradeoff space: slower to stand up, but the right choice once a
system is going to production with a real UX bar and a dedicated frontend team."*

### Streamlit

Streamlit is a general-purpose Python app framework — it reruns your whole script top-to-bottom on
every user interaction, using `st.session_state` to persist anything across those reruns (like
conversation history).

```python
import streamlit as st
from langchain_core.messages import HumanMessage, AIMessage

st.title("Internal Assistant — Prototype")

if "messages" not in st.session_state:
    st.session_state.messages = []

for msg in st.session_state.messages:
    role = "user" if isinstance(msg, HumanMessage) else "assistant"
    with st.chat_message(role):
        st.markdown(msg.content)

if user_input := st.chat_input("Ask a question..."):
    st.session_state.messages.append(HumanMessage(content=user_input))
    with st.chat_message("user"):
        st.markdown(user_input)

    with st.chat_message("assistant"):
        placeholder = st.empty()
        full_response = ""
        for chunk in chain.stream({"question": user_input, ...}):  # chain from Chapter 2
            full_response += chunk
            placeholder.markdown(full_response + "▌")
        placeholder.markdown(full_response)

    st.session_state.messages.append(AIMessage(content=full_response))
```

Two things worth understanding, not just copying: `st.session_state` exists because Streamlit's
execution model reruns the entire script on every widget interaction — without it, conversation history
would reset on every keystroke's rerun (the Streamlit-specific flavor of "the model has no memory, the
app owns memory" from Chapter 1). And `st.empty()` plus incremental `.markdown()` calls is how you
render a streaming LCEL chain token-by-token in a Streamlit UI, connecting the backend streaming
capability from Chapter 2 to a perceptibly faster prototype UX.

### Chainlit

Chainlit is purpose-built for LLM chat apps specifically, with first-class async support, native
streaming, and built-in LangChain integration:

```python
import chainlit as cl

@cl.on_chat_start
async def start():
    cl.user_session.set("history", [])

@cl.on_message
async def main(message: cl.Message):
    history = cl.user_session.get("history")
    msg = cl.Message(content="")
    await msg.send()

    async for chunk in chain.astream({"question": message.content, "history": history}):
        await msg.stream_token(chunk)

    await msg.update()
    history.append((message.content, msg.content))
    cl.user_session.set("history", history)
```

`cl.user_session` is Chainlit's equivalent of `st.session_state`. Native async
(`@cl.on_message async def ...`, `msg.stream_token`) fits naturally with LangChain's `astream`/`ainvoke`
`Runnable` methods (Chapter 2). Chainlit also renders intermediate steps (a retrieval step, a tool call)
as collapsible UI elements automatically via its LangChain callback integration, and ships built-in
thumbs-up/down feedback collection tied to a persisted conversation ID — both genuinely useful during a
POC when you want fast, low-effort signal on whether a demo is landing.

### Where this genuinely fits in the story

Streamlit and Chainlit remain the right answer for: standing up a same-day demo for a client
stakeholder meeting, an internal tool with no dedicated frontend budget, or a personal project like the
Text-to-Math solver (Chapter 5), which explicitly shipped as a Streamlit app because that *was* its
production surface — a personal project has no separate frontend team to hand off to. The distinction
that matters is: **who is the audience, and is there a frontend team on the other side of a handoff.**
For a POC or an internal/personal tool, there usually isn't, and Python-native UI tooling is the right
call. For a production, customer-facing banking system, there is, and Part 2 is what that looks like.

## Part 2 — The real production frontend: `react-service`

### What it is, and why it's a separate service

`react-service` is a React single-page application (SPA), built and deployed as its **own microservice**
— its own codebase, its own Azure App Service (or Static Web App) hosting target, its own release
pipeline — entirely separate from the FastAPI backend that owns the LLM orchestration, retrieval, and
conversation-state logic (Chapters 2–3). The backend exposes a Chat API; `react-service` is a client of
that API, not a module deployed inside the same process or App Service instance as the backend.

This is a deliberate architectural boundary, not an accident of how the project happened to be built,
and it's worth being able to explain *why* a consultancy would draw the line here:

- **Team and access-boundary separation.** Frontend developers building and iterating on `react-service`
  don't need deploy access to the backend, its Azure OpenAI credentials, its Azure AI Search
  configuration, or its database connection strings — and vice versa, backend engineers don't need to be
  in the critical path for a CSS fix or a new UI component. In a regulated-client engagement, narrowing
  who can deploy what is a real security and change-management benefit, not just a convenience.
- **Independent release cadence.** A wording tweak, a new quick-reply button, or a design refresh on
  `react-service` can ship without touching, redeploying, or re-testing the backend's LLM pipeline at
  all — and conversely, a prompt-engineering change or a new retrieval parameter on the backend doesn't
  require a frontend rebuild. Coupling the two into one deployable would mean every change, regardless of
  which layer it touched, carried the blast radius and testing burden of the whole system.
- **Different scaling profiles.** `react-service` serves static assets (the compiled JS/CSS bundle) plus
  thin, cheap API calls — it scales like any static frontend, mostly bound by request count and CDN/edge
  caching. The backend is bound by Azure OpenAI TPM/RPM quotas and LLM-orchestration compute (Chapter 3)
  — a completely different scaling story. Running them as one service would mean scaling decisions for
  one workload (say, a traffic spike on the marketing-adjacent parts of the UI) unnecessarily drag along
  scaling for the other (expensive LLM compute), and vice versa.
- **Faster frontend iteration.** Because `react-service` isn't in the same deploy unit as the backend, a
  frontend team can iterate and redeploy multiple times a day without needing a backend release window,
  smoke tests against the LLM pipeline, or coordination with whoever owns the Azure OpenAI deployment.

### The API contract: streaming chat responses

`react-service` and the backend communicate over a defined HTTP API. The chat endpoint specifically
needs to **stream** tokens back to the browser — the same time-to-first-token perceived-latency argument
from Chapter 2/Chapter 3 applies just as much to a React frontend as it did to a Chainlit/Streamlit one.
The two realistic mechanisms are **Server-Sent Events (SSE)** over a chunked HTTP response, or a
**WebSocket** connection; SSE is the lighter-weight, more common choice for a one-directional
server-to-client token stream like this (a WebSocket buys you bidirectional messaging you don't actually
need for a request/response chat turn), so that's the concrete example below.

A sample event shape, one JSON object per SSE `data:` line:

```json
{"type": "token", "content": "Ref"}
{"type": "token", "content": "unds"}
{"type": "citation", "source": "Refund_Policy_v2.pdf", "chunk_id": "doc_84a3::chunk0"}
{"type": "done", "conversation_id": "b3f1c9..."}
```

`type: "token"` events are appended to the in-progress answer as they arrive; `type: "citation"` events
let the UI render a source reference alongside the answer (tying back to the RAG grounding story in
Chapter 3); `type: "done"` signals the stream is complete and the UI can stop showing a "typing"
indicator.

**Backend side — FastAPI/Starlette `StreamingResponse`:**

```python
import json
from fastapi import FastAPI, Depends
from fastapi.responses import StreamingResponse

app = FastAPI()

async def event_stream(question: str, conversation_id: str):
    async for chunk in chain.astream({"question": question, ...}):  # LCEL chain, Chapter 2
        yield f"data: {json.dumps({'type': 'token', 'content': chunk})}\n\n"
    yield f"data: {json.dumps({'type': 'done', 'conversation_id': conversation_id})}\n\n"

@app.post("/api/chat/stream")
async def chat_stream(payload: ChatRequest, user=Depends(get_current_user)):
    return StreamingResponse(
        event_stream(payload.question, payload.conversation_id),
        media_type="text/event-stream",
    )
```

**Frontend side — `react-service` consuming the stream.** The browser's native `EventSource` API is
GET-only and can't attach a custom `Authorization` header, which doesn't fit a bearer-token-authenticated
POST endpoint — so the realistic implementation is a `fetch()` call reading the response body as a
stream, manually parsing SSE-formatted lines, rather than a literal `EventSource`:

```jsx
async function streamChat(question, conversationId, accessToken, onToken, onDone) {
  const response = await fetch("/api/chat/stream", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${accessToken}`,
    },
    body: JSON.stringify({ question, conversation_id: conversationId }),
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    const events = buffer.split("\n\n");
    buffer = events.pop(); // keep any partial event for the next chunk

    for (const raw of events) {
      if (!raw.startsWith("data: ")) continue;
      const event = JSON.parse(raw.slice(6));
      if (event.type === "token") onToken(event.content);
      if (event.type === "done") onDone(event.conversation_id);
    }
  }
}
```

### Auth: MSAL.js in the browser, the same Azure AD app registration from Chapter 3/Course 05

`react-service` doesn't invent its own auth model — it uses **MSAL.js** (`@azure/msal-react` /
`@azure/msal-browser`) to sign the user in against the **same Azure AD tenant and app registration
family** covered in Chapter 3 and Course 05's dedicated RBAC/MSAL chapter, just from the browser side of
the flow instead of the backend side. Concretely: `react-service` is registered as a **public client
(SPA)** app registration using the authorization-code flow with PKCE, while the FastAPI backend is
exposed as the **resource server** with its own scope (e.g. `api://<backend-app-id>/Chat.Access`) that
the SPA registration requests access to.

```jsx
import { useMsal } from "@azure/msal-react";

function useBackendAccessToken() {
  const { instance, accounts } = useMsal();

  return async function getAccessToken() {
    const response = await instance.acquireTokenSilent({
      scopes: ["api://<backend-app-registration-id>/Chat.Access"],
      account: accounts[0],
    });
    return response.accessToken; // attached as a bearer token on every backend call
  };
}
```

That token is attached as an `Authorization: Bearer <token>` header on every call `react-service` makes
to the backend Chat API — including the streaming endpoint above. On the backend side, this is exactly
the token the Azure AD-based authentication described in Chapter 3 (and implemented in depth in Course 05's chapter 04) is validating and extracting role claims from — `react-service` doesn't change *what*
gets validated, it's simply the browser-side actor acquiring the token in the first place, via
`acquireTokenSilent` (falling back to an interactive redirect/popup if no cached/refreshable token
exists) instead of a server-side authorization-code exchange.

### CORS: two separate origins, one authenticated API

Because `react-service` and the backend are separate deployments, they are — by definition — **separate
origins** from the browser's point of view (different hostnames, or at minimum different subdomains/App
Service instances), which means every call from `react-service` to the backend is a cross-origin
request, subject to the browser's CORS (Cross-Origin Resource Sharing) policy. The backend has to
explicitly allow it:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://react-service.internal.example.com"],  # explicit allow-list, never "*"
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
)
```

Two things worth being precise about in an interview: an explicit **origin allow-list**, not a wildcard
(`*`), is required the moment `allow_credentials=True` is set (the CORS spec disallows combining a
wildcard origin with credentialed requests, and even where it didn't, an authenticated banking API has
no business accepting cross-origin calls from an arbitrary origin) — and the streaming `POST` endpoint
above triggers a CORS **preflight** `OPTIONS` request from the browser first (since it carries a custom
`Authorization` header and a JSON content type), which the backend's CORS middleware has to answer
correctly before the browser will even send the real request.

## Tying It Back

If asked "what did the actual user-facing product look like in production," the accurate answer is:
`react-service`, a standalone React SPA deployed as its own microservice, calling the FastAPI backend's
Chat API over a streamed (SSE) connection, authenticated via MSAL.js against the same Azure AD app
registration family covered in Chapter 3/Course 05, with CORS explicitly configured for the two
services' separate origins. That separation — frontend and backend as independently deployable,
independently scaled services with a clean access boundary between the teams that own them — is itself
a good answer to "how would you architect a production LLM chat product," independent of the LLM
specifics. Chainlit and Streamlit (Part 1) remain true, resume-worthy skills for exactly the phase where
that separation isn't yet justified: a same-day POC, an internal tool, or a personal project like the
Text-to-Math solver in Chapter 5 — the honest story is "both are real," not "one replaced the other."
