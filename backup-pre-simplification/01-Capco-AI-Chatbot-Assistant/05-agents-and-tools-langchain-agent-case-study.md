# 05 — LangChain Agents & Tools: Case Study (Text-to-Math Solver)

## Why this chapter matters

This chapter folds in the personal project from the resume: *"Text to Math Problem solver — LangChain
AI Agent with Tools to Solve Math Problems. Streamlit for interactive app and slick UI. Chatbot with
math-solving capabilities. Works with Gemma2-9b via Groq API."* It's included here because it
demonstrates the same LangChain-agent skillset a production chatbot needs whenever a question
requires *doing something* (a calculation, a lookup, an API call) rather than just retrieving and
summarizing text — a natural extension of the Capco chatbot's architecture, and a great worked
example because it's small enough to reason about completely.

## Why Plain LLM Calls Aren't Enough for Math

Recall from Chapter 1: an LLM is a next-token predictor trained to produce plausible text — it has no
built-in arithmetic unit. Ask a raw LLM "what is 847 × 293?" and it will generate tokens that *look*
like a confident numeric answer, because that's what plausible completions of that pattern look like
in its training data — with no guarantee the digits are actually correct. This is a clean,
easy-to-demonstrate case of the hallucination failure mode from Chapter 1, and it's exactly why the
project exists: wrap the LLM with actual tools that can compute a correct answer, and have the LLM
decide *when* to call them and *how* to interpret the result.

## LangChain Agents: The Core Idea

Where an LCEL chain (Chapter 2) is a **fixed, predetermined sequence** of steps (prompt → LLM →
parser, always in that order), an **agent** is a loop where the LLM itself decides, at each step,
which action to take next — including whether to call a tool, and which one — based on the evolving
state of the conversation. The agent framework provides:

- **Tools**: Python functions (or LangChain-provided integrations) with a name, a description, and an
  input schema, that the agent can choose to invoke.
- **An agent executor**: the loop that repeatedly asks the LLM "given the conversation and any tool
  results so far, what's the next action?", executes whatever the LLM decides, feeds the result back
  in, and repeats until the LLM decides it has a final answer.

### The ReAct Pattern

ReAct (Reason + Act) is the specific prompting pattern most LangChain agents implement: at each step,
the model is prompted to produce a **Thought** (reasoning about what to do next), an **Action** (which
tool to call and with what input), then the framework executes that action and feeds back an
**Observation** (the tool's result), and the loop continues until the model emits a **Final Answer**.
This directly uses the chain-of-thought mechanism from Chapter 1 — the "Thought" step gives the model
room to reason in tokens before committing to an action, which measurably improves the quality of the
tool-selection decision compared to jumping straight to an action.

A trace for "What is 15% of 340, then add 12?" might look like:

```
Thought: I need to calculate 15% of 340 first, then add 12. I should use the calculator tool.
Action: calculator
Action Input: 0.15 * 340
Observation: 51.0
Thought: Now I add 12 to 51.0.
Action: calculator
Action Input: 51.0 + 12
Observation: 63.0
Thought: I have the final answer.
Final Answer: 63
```

## Defining Tools

In modern LangChain, a tool is typically a plain Python function decorated with `@tool`, where the
docstring becomes the description the LLM uses to decide *when* this tool is relevant — this is worth
emphasizing in an interview: **prompt engineering doesn't stop at the system prompt; the tool
descriptions are prompts too**, and a vague description leads directly to the agent picking the wrong
tool or not using a tool when it should.

```python
from langchain_core.tools import tool

@tool
def calculator(expression: str) -> str:
    """Evaluate a basic arithmetic expression, e.g. '0.15 * 340 + 12'. 
    Use this whenever the problem requires a numeric calculation."""
    try:
        # a real implementation should use a safe expression evaluator,
        # not raw eval() -- see the notebook for a safer approach
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"Error: {e}"

@tool
def word_problem_parser(problem: str) -> str:
    """Restate a math word problem as a plain arithmetic expression, without solving it."""
    # in the real project, this itself could be an LLM call --
    # a chain nested inside a tool used by an agent
    ...
```

A "text to math" solver plausibly used two tools like this: one to translate the natural-language
word problem into a formal expression (leaning on the LLM's language understanding), and a
calculator tool to actually evaluate it (leaning on deterministic code, not the LLM, for the part
that must be numerically correct) — the general pattern of **using the LLM for what it's good at
(language understanding, decomposition) and code for what it's good at (exact computation)**, which
generalizes to any agent design question you might get in an interview.

## Wiring Up the Agent

```python
from langchain.agents import create_react_agent, AgentExecutor
from langchain_core.prompts import PromptTemplate
from langchain_groq import ChatGroq

llm = ChatGroq(model="gemma2-9b-it", temperature=0)  # see Groq/Gemma2 note below

react_prompt = PromptTemplate.from_template(
    "Answer the following math word problem as best you can. "
    "You have access to these tools:\n{tools}\n\n"
    "Use this format:\nThought: ...\nAction: {tool_names}\n"
    "Action Input: ...\nObservation: ...\n... (repeat)\nFinal Answer: ...\n\n"
    "Question: {input}\n{agent_scratchpad}"
)

tools = [calculator, word_problem_parser]
agent = create_react_agent(llm, tools, react_prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True, max_iterations=6)

result = executor.invoke({"input": "A train travels 60 miles in 45 minutes. What is its speed in mph?"})
print(result["output"])
```

The `agent_scratchpad` is where the running Thought/Action/Observation history accumulates across
loop iterations — it's the mechanism that lets the LLM "see" its own previous tool calls and their
results within the same problem-solving turn, the same "everything must be in the context window"
principle from Chapter 1, just applied within a single agent run instead of across conversation
turns.

## The Groq / Gemma2-9b Detail

The project specifically ran on **Gemma2-9b via the Groq API**. This is worth being able to explain
because it's a genuinely different point in the design space from Azure OpenAI:

- **Gemma2-9b** is a relatively small (9-billion parameter), open-weight model from Google — capable
  enough for structured tasks like tool selection and simple reasoning, at a fraction of the cost and
  latency of a frontier model.
- **Groq** is notable for extremely fast inference, achieved via custom LPU (Language Processing
  Unit) hardware rather than GPUs — Groq-hosted open-weight models routinely generate at hundreds of
  tokens/second, which matters a lot for an agent loop specifically, since each ReAct iteration is a
  full LLM round trip: a slow model means a multi-step agent (Thought → Action → Observation, several
  times) compounds latency badly, whereas Groq's speed keeps even a 4-5-step agent loop feeling
  responsive in a UI.
- This also demonstrates provider-agnostic thinking with LangChain's abstraction (Chapter 2) — the
  same `ChatGroq` wrapper slots into the identical `Runnable`/agent interfaces as `AzureChatOpenAI`,
  which is a good concrete example if asked "how would you compare/swap LLM providers."

## Tying It Back

If asked "tell me about a project where you built an agent, not just a chatbot," this is it: an LLM
deciding when to invoke a calculator tool to solve math word problems reliably, using the ReAct
pattern for transparent step-by-step reasoning, a Streamlit UI for interactivity, and Gemma2-9b on
Groq for fast, low-cost inference suited to a lightweight tool-use loop. It also gives you a concrete
answer to "when would you use an agent instead of a simple LCEL chain?" — whenever the *sequence of
steps isn't known in advance* and needs to be decided dynamically based on the specific input, which
is exactly the difference between the fixed `prompt | llm | parser` chain from Chapter 2 and the
agent executor's loop here.
