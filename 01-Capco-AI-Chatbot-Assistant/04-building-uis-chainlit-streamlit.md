# 04 — Building Chatbot UIs: Streamlit & Chainlit

## Why this chapter matters

Both Streamlit and Chainlit appear in the candidate's skill list, and both are exactly the tools a
consultancy reaches for to get a working chatbot demo in front of a client fast, before investing in
a custom frontend. Interviewers asking about "the chatbot" often mean the whole user-facing
experience, not just the backend chain — this chapter gives you the vocabulary for that half of the
story, and it's also the layer the personal Text-to-Math project (Chapter 5) actually ships as.

## Why Rapid UI Frameworks Matter for a Consultancy

A management/technology consultancy engagement typically runs through a pilot/POC phase before a
client commits budget to a production build-out. In that phase, the goal is to prove the concept
works and get real user feedback — not to hand-build a React frontend with authentication, custom
components, and a design system. Streamlit and Chainlit both let a backend-focused Python developer
ship a usable, presentable chat UI in hours, entirely in Python, with no separate frontend codebase
to maintain. That speed is the whole value proposition, and it's worth stating explicitly in an
interview: choosing Streamlit/Chainlit was very likely a deliberate "move fast for the pilot" call,
not a lack of frontend skill.

## Streamlit

Streamlit is a general-purpose Python app framework — reruns your whole script top-to-bottom on every
user interaction, using `st.session_state` to persist anything across those reruns (like
conversation history).

```python
import streamlit as st
from langchain_core.messages import HumanMessage, AIMessage

st.title("Acme Bank Internal Assistant")

if "messages" not in st.session_state:
    st.session_state.messages = []

# render prior turns
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
        # chain from Chapter 2, streaming tokens
        for chunk in chain.stream({"question": user_input, ...}):
            full_response += chunk
            placeholder.markdown(full_response + "▌")
        placeholder.markdown(full_response)

    st.session_state.messages.append(AIMessage(content=full_response))
```

Two things worth understanding, not just copying:

- **`st.session_state`** exists because Streamlit's execution model reruns the entire script on every
  widget interaction — without `session_state`, `st.session_state.messages` would reset to empty on
  every single keystroke's rerun. This is the Streamlit-specific analogue of the "the model has no
  memory, the app owns memory" point from Chapter 1 — Streamlit itself *also* has no memory between
  reruns unless you explicitly store it.
- **`st.empty()` + incremental `.markdown()` calls** is how you render a streaming LCEL chain
  (Chapter 2) token-by-token in the UI instead of waiting for the full response — directly connecting
  the backend streaming capability to a perceptibly faster UI.

Streamlit also ships built-in feedback widgets (`st.feedback`, or simple 👍/👎 buttons via
`st.button`) that are commonly wired up to log user satisfaction per response — useful both as a
product signal and as input to the kind of monitoring covered in Course 02.

## Chainlit

Chainlit is purpose-built for LLM chat apps specifically (rather than Streamlit's general-purpose
app model), with first-class async support, native streaming, and built-in LangChain integration —
often a better fit once you're specifically building a *chat* interface rather than a general
dashboard-plus-chat hybrid.

```python
import chainlit as cl

@cl.on_chat_start
async def start():
    cl.user_session.set("history", [])

@cl.on_message
async def main(message: cl.Message):
    history = cl.user_session.get("history")

    msg = cl.Message(content="")
    await msg.send()  # send an empty message we'll stream into

    async for chunk in chain.astream({"question": message.content, "history": history}):
        await msg.stream_token(chunk)

    await msg.update()
    history.append((message.content, msg.content))
    cl.user_session.set("history", history)
```

Notable differences from Streamlit worth being able to articulate:

- **`cl.user_session`** is Chainlit's equivalent of `st.session_state`, scoped per chat session —
  same underlying need (the app owns conversation state), different API.
- **Native async** (`@cl.on_message async def ...`, `msg.stream_token`) fits naturally with
  LangChain's `astream`/`ainvoke` async `Runnable` methods from Chapter 2, which matters for a
  backend serving multiple concurrent users without blocking on each LLM call.
- Chainlit renders **intermediate steps** (e.g., a retrieval step, a tool call) as collapsible UI
  elements automatically when using its LangChain callback integration — genuinely useful for a
  chatbot where you want the user (or an internal reviewer) to see *what document was retrieved* to
  ground an answer, tying back to the RAG grounding story from Chapter 3.
- Chainlit has built-in feedback collection (thumbs up/down on each message, tied to a persisted
  conversation ID) out of the box, which is closer to production-monitoring-ready than Streamlit's
  more manual approach.

## Choosing Between Them

| | Streamlit | Chainlit |
|---|---|---|
| Best for | General data apps, dashboards, quick internal tools with a chat *component* | Purpose-built chat/conversational UIs |
| Session model | `st.session_state`, rerun-the-script-on-every-interaction | `cl.user_session`, persistent async event handlers |
| Streaming | Manual (`st.empty()` + incremental updates) | Native (`stream_token`) |
| LangChain integration | Manual wiring | Built-in callback handler, auto-renders intermediate steps |
| Typical use in this project | Fast POC UI for the Capco chatbot backend | Equally plausible choice for the same POC; also the natural fit for a pure chatbot |

In an interview, the honest answer is: both were used across the resume ("chainlit, streamlit" are
both listed), which most likely reflects using whichever fit a given engagement's timeline and
audience — Streamlit when the deliverable needed to sit alongside other data views, Chainlit when the
deliverable was purely conversational.

## Tying It Back

This chapter is your answer to "what did the actual user-facing product look like?" — a chat-style
interface, built fast in Python without a separate frontend team, streaming tokens from the LCEL
chain (Chapter 2) as they're generated from Azure OpenAI (Chapter 3), with lightweight feedback
capture. It's also directly the delivery mechanism for the personal Text-to-Math project discussed
next in Chapter 5, which explicitly used "Streamlit for interactive app and slick UI."
