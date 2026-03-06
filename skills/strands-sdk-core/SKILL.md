---
name: strands-sdk-core
description: "Use when working with Strands-agents SDK for the first time, importing Agent or BedrockModel, initializing an agent, invoking an agent with a prompt, or asking about basic agent setup. Triggers on: from strands import Agent, BedrockModel, strands-agents, agent initialization, basic invocation."
---

# Core — Strands SDK

## Overview

The Strands-agents SDK lets you build AI agents powered by LLMs. An `Agent` wraps a model, an optional system prompt, and an optional set of tools. You call it like a function with a string prompt and get back a structured result object. The agent automatically manages conversation history across turns, routes tool calls, accumulates tool outputs, and returns a final text response.

Strands is model-agnostic: its `BedrockModel` class connects to AWS Bedrock, but the same `Agent` interface works with any conforming model adapter.

---

## Quick Reference

| Concept | Import | Example |
|---|---|---|
| Agent | `from strands import Agent` | `agent = Agent(model=model)` |
| BedrockModel | `from strands.models import BedrockModel` | `BedrockModel(model_id="...")` |
| Synchronous invoke | — | `result = agent("your prompt")` |
| Async invoke | — | `result = await agent.invoke_async("prompt")` |
| System prompt | constructor | `Agent(model=model, system_prompt="You are...")` |
| Add tools | constructor | `Agent(model=model, tools=[my_tool])` |
| Read response text | — | `str(result)` or `result.message` |
| Inspect history | — | `agent.messages` |
| Inspect model | — | `agent.model` |
| Inspect tools | — | `agent.tools` |

---

## Installation

```bash
pip install strands-agents
# Optional: install built-in tool collections
pip install strands-agents-tools
```

Verify the installation:

```python
import strands
print(strands.__version__)
```

---

## Basic Usage

### Minimal synchronous agent

```python
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

agent = Agent(model=model)
result = agent("What is the capital of France?")
print(str(result))
```

### Agent with a system prompt

A system prompt defines the agent's persona, tone, constraints, and default behavior. It is prepended to every conversation turn automatically.

```python
agent = Agent(
    model=model,
    system_prompt=(
        "You are a senior Python engineer. "
        "Reply concisely. Prefer idiomatic Python 3.11+. "
        "When showing code, always include type hints."
    )
)

result = agent("How do I flatten a nested list?")
print(str(result))
```

### Async invocation

Use `invoke_async` inside any async context (FastAPI handlers, async scripts, Jupyter notebooks). Calling the synchronous `agent(prompt)` from inside a running event loop will block the loop.

```python
import asyncio
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)
agent = Agent(model=model, system_prompt="You are a helpful assistant.")

async def ask(prompt: str) -> str:
    result = await agent.invoke_async(prompt)
    return str(result)

answer = asyncio.run(ask("Summarise the water cycle in two sentences."))
print(answer)
```

### FastAPI integration (async)

```python
from fastapi import FastAPI
from strands import Agent
from strands.models import BedrockModel

app = FastAPI()

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)
agent = Agent(model=model, system_prompt="You are a customer support assistant.")

@app.post("/chat")
async def chat(body: dict):
    prompt = body.get("message", "")
    result = await agent.invoke_async(prompt)
    return {"response": str(result)}
```

---

## Agent Constructor Parameters

| Parameter | Type | Default | Purpose |
|---|---|---|---|
| `model` | `BedrockModel` or provider | required | The LLM backend that processes prompts |
| `system_prompt` | `str` | `None` | Persona / role instruction prepended to every turn |
| `tools` | `list` | `[]` | Callable tools the agent may invoke during a turn |
| `conversation_manager` | `ConversationManager` | `SlidingWindowConversationManager` | Controls how message history is retained and truncated |
| `hooks` | `list` | `[]` | Lifecycle callback hooks (before/after invoke, tool call, etc.) |
| `guardrail` | `Guardrail` | `None` | Input/output safety layer (content filtering, PII, etc.) |
| `trace_attributes` | `dict` | `{}` | Key-value metadata forwarded to observability/tracing backends |
| `max_parallel_tools` | `int` | `1` | Number of tool calls that may execute concurrently |

---

## BedrockModel Configuration

`BedrockModel` wraps an AWS Bedrock model. All standard Bedrock converse-API parameters are supported.

```python
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="ap-south-1",      # defaults to AWS_REGION env var
    temperature=0.3,               # 0.0 = deterministic, 1.0 = creative
    max_tokens=4096,               # maximum tokens in the response
    top_p=0.9,
)
```

Credentials are resolved by the standard boto3 chain: environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`), `~/.aws/credentials`, or an instance profile. For Bedrock, the IAM principal needs the `bedrock:InvokeModel` permission on the target model ARN.

---

## Agent Lifecycle — How an Invocation Works Internally

Understanding the invocation lifecycle helps you debug unexpected behavior and design more reliable agents.

1. **Prompt ingestion** — The string you pass to `agent(prompt)` is appended to the conversation history as a `user` message.
2. **System prompt injection** — The configured `system_prompt` is prepended as a `system` message (handled transparently; you never add it manually).
3. **Model call** — The full message list is sent to `BedrockModel.invoke()`, which calls the Bedrock Converse API.
4. **Response parsing** — The SDK parses the model response. If the model returns plain text, that text is wrapped in an `AgentResult` and returned immediately (step 7).
5. **Tool dispatch** — If the model returns one or more tool-use blocks, the SDK matches each tool name to a registered callable, executes them (sequentially or in parallel depending on `max_parallel_tools`), and collects results.
6. **Tool result re-injection** — Tool results are appended to the conversation history as `tool` messages, and the model is called again with the updated history. Steps 4–6 repeat until the model produces a plain-text final answer or a stop condition is reached.
7. **Result return** — The final `AgentResult` is returned. History is automatically updated so the next call to `agent(...)` continues the conversation.

```
agent("prompt")
    │
    ▼
Append user message to history
    │
    ▼
Call model with full history
    │
    ├─── plain text ──────────────────► Return AgentResult
    │
    └─── tool calls
            │
            ▼
        Execute tools → append tool results to history
            │
            ▼
        Call model again  (loop until plain text)
```

---

## Tools

### Defining a tool

Any Python callable decorated with `@tool` (or matching the tool schema interface) can be registered. The decorator uses the function's type annotations and docstring to generate the JSON schema the model uses for tool selection.

```python
from strands import Agent, tool
from strands.models import BedrockModel

@tool
def get_weather(city: str) -> str:
    """Return the current weather for a city.

    Args:
        city: The city name, e.g. 'London'.

    Returns:
        A plain-text weather description.
    """
    # Replace with a real API call in production
    return f"The weather in {city} is 22°C and sunny."

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

agent = Agent(model=model, tools=[get_weather])
result = agent("What is the weather in Tokyo?")
print(str(result))
```

### Multiple tools

```python
@tool
def add(a: int, b: int) -> int:
    """Add two integers."""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two integers."""
    return a * b

agent = Agent(model=model, tools=[add, multiply])
result = agent("What is (3 + 4) * 5?")
print(str(result))  # The agent will call add then multiply
```

### Async tool

```python
import httpx
from strands import tool

@tool
async def fetch_url(url: str) -> str:
    """Fetch the text content of a URL.

    Args:
        url: The full URL to fetch.
    """
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, timeout=10)
        return resp.text[:2000]

agent = Agent(model=model, tools=[fetch_url])
result = await agent.invoke_async("Summarise https://example.com")
print(str(result))
```

---

## Structuring Agents for Different Use Cases

### Stateless Q&A agent (no memory between calls)

Create a fresh `Agent` per request. Each instance starts with an empty history.

```python
def answer_once(question: str) -> str:
    fresh_agent = Agent(model=model, system_prompt="Answer concisely.")
    return str(fresh_agent(question))
```

### Stateful conversational agent (memory across turns)

Reuse the same `Agent` instance. The SDK accumulates messages automatically.

```python
agent = Agent(model=model, system_prompt="You are a friendly tutor.")

while True:
    user_input = input("You: ")
    if user_input.lower() in ("exit", "quit"):
        break
    result = agent(user_input)
    print(f"Tutor: {str(result)}\n")
```

### Per-session agent (web app pattern)

Store one `Agent` per session ID. This gives each user their own history while sharing the same model object.

```python
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

sessions: dict[str, Agent] = {}

def get_agent(session_id: str) -> Agent:
    if session_id not in sessions:
        sessions[session_id] = Agent(
            model=model,
            system_prompt="You are a helpful shopping assistant."
        )
    return sessions[session_id]

async def handle_message(session_id: str, message: str) -> str:
    agent = get_agent(session_id)
    result = await agent.invoke_async(message)
    return str(result)
```

### Specialist pipeline (multiple agents)

Chain agents where each handles a distinct responsibility.

```python
researcher = Agent(
    model=model,
    system_prompt="You extract key facts from raw text. Be concise."
)
writer = Agent(
    model=model,
    system_prompt="You turn bullet-point facts into polished prose."
)

raw_text = "..."  # raw source material
facts = researcher(f"Extract key facts from this text:\n{raw_text}")
article = writer(f"Write a short article from these facts:\n{str(facts)}")
print(str(article))
```

---

## Context Window Management

Every model has a finite context window (measured in tokens). As a conversation grows, the accumulated messages can exceed that limit and cause a `ValidationException` or silently truncated output.

### Default: SlidingWindowConversationManager

By default, Strands uses a `SlidingWindowConversationManager` that keeps only the most recent N turns of conversation, automatically dropping the oldest messages when the window is full. The system prompt is always retained.

```python
from strands import Agent
from strands.conversation_manager import SlidingWindowConversationManager

agent = Agent(
    model=model,
    conversation_manager=SlidingWindowConversationManager(window_size=20)
)
```

### Manually resetting history

If you want a hard reset (e.g. between topics in the same session):

```python
agent.messages.clear()
```

### Inspecting token usage

After each invocation the result object carries metadata:

```python
result = agent("Explain neural networks.")
# Token usage is available in the result metadata (model-dependent)
print(result.usage)  # e.g. {'inputTokens': 150, 'outputTokens': 320}
```

---

## Inspecting Agent State

### Messages (conversation history)

`agent.messages` is a plain Python list of dicts following the Bedrock Converse message schema.

```python
result = agent("Hello, who are you?")

for msg in agent.messages:
    role = msg["role"]
    # Content can be a string or a list of content blocks
    content = msg["content"]
    print(f"[{role}] {content}")
```

### Model

```python
print(agent.model)                   # BedrockModel instance
print(agent.model.model_id)          # e.g. 'anthropic.claude-haiku-4-5-20251001-v1:0'
```

### Tools

```python
print(agent.tools)                   # list of registered tool callables
print([t.__name__ for t in agent.tools])
```

### System prompt

```python
print(agent.system_prompt)
```

---

## Real-World Usage Patterns

### Pattern 1: Structured output via JSON

Instruct the agent to respond in JSON and parse the result.

```python
import json
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

agent = Agent(
    model=model,
    system_prompt=(
        "You are a data extraction assistant. "
        "Always respond with valid JSON only. No prose."
    )
)

result = agent(
    'Extract name, age, and city from: "Alice is 30 and lives in Berlin."'
)

data = json.loads(str(result))
print(data)  # {"name": "Alice", "age": 30, "city": "Berlin"}
```

### Pattern 2: Document summarisation

```python
def summarise(document: str, max_words: int = 150) -> str:
    agent = Agent(
        model=model,
        system_prompt=f"Summarise documents in at most {max_words} words."
    )
    result = agent(f"Summarise the following:\n\n{document}")
    return str(result)
```

### Pattern 3: Retry on empty or malformed response

```python
import time

def reliable_ask(agent: Agent, prompt: str, retries: int = 3) -> str:
    for attempt in range(retries):
        result = agent(prompt)
        text = str(result).strip()
        if text:
            return text
        time.sleep(1)
    raise RuntimeError("Agent returned empty response after retries")
```

### Pattern 4: Streaming output (if supported by backend)

```python
# Some model adapters support streaming; check the adapter docs.
# When streaming is enabled, partial tokens are yielded as they arrive.
async for chunk in agent.stream_async("Write a short poem about clouds."):
    print(chunk, end="", flush=True)
```

### Pattern 5: Tool with context injection

Pass context to a tool via closure, avoiding globals.

```python
def make_db_lookup_tool(db_connection):
    @tool
    def lookup_user(user_id: str) -> str:
        """Look up a user record by ID.

        Args:
            user_id: The unique user identifier.
        """
        row = db_connection.execute(
            "SELECT name, email FROM users WHERE id = ?", (user_id,)
        ).fetchone()
        return str(dict(row)) if row else "User not found"

    return lookup_user

db = get_db_connection()  # your DB connection
agent = Agent(model=model, tools=[make_db_lookup_tool(db)])
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Printing `result` directly | Gets object repr like `<AgentResult ...>` | Use `str(result)` or `result.message` |
| Calling `agent(prompt)` in async context | Blocks the event loop; potential deadlock | Use `await agent.invoke_async(prompt)` |
| Model ID typo | `ValidationException: model identifier` | Copy exact ID from AWS Bedrock console |
| Missing `AWS_REGION` | `NoRegionError` or wrong region endpoint | Set `region_name` in `BedrockModel` or export `AWS_REGION` |
| Using SSO role with Bedrock Deny policy | `AccessDeniedException` | Switch to IAM user credentials; verify with `aws sts get-caller-identity` |
| Creating a new `Agent` per turn | History lost between messages; no multi-turn context | Reuse the same `Agent` instance across turns |
| Registering a tool without a docstring | Model cannot infer tool purpose; poor or no tool calls | Add a clear docstring with `Args:` and `Returns:` sections |
| Forgetting `await` on `invoke_async` | Returns a coroutine object, not a result | Always `await agent.invoke_async(...)` |
| Passing `system_prompt` text as first user message | System prompt ends up in conversation history; doubled context | Pass via the `system_prompt` constructor parameter, never as a user message |
| Sharing one `Agent` instance across concurrent async tasks | Race condition on shared message history | Create one `Agent` per concurrent task or use per-session instances |
| Not clearing messages between unrelated sessions | Previous user's history leaks into new session | Call `agent.messages.clear()` or create a fresh `Agent` per session |
| Overloading tool count (10+ tools) | Model struggles to choose the right tool; hallucinated calls | Split into focused sub-agents each with 2–5 relevant tools |
| Very large system prompt | Wastes tokens every turn; can hit context limit | Keep system prompts under ~500 words; move static data to tool results |

---

## Troubleshooting

| Error / Symptom | Likely Cause | Resolution |
|---|---|---|
| `ModuleNotFoundError: No module named 'strands'` | Package not installed | `pip install strands-agents` |
| `ValidationException: model identifier` | Wrong `model_id` string or unsupported region | Copy exact model ID from AWS Bedrock console; check `region_name` |
| `AccessDeniedException` on Bedrock | IAM principal lacks `bedrock:InvokeModel`; SSO Deny policy | Use IAM user credentials; attach Bedrock invoke policy |
| `NoCredentialsError` | No AWS credentials found | Set `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` or configure `~/.aws/credentials` |
| `EndpointResolutionError` | Wrong region or region not in boto3 chain | Pass `region_name` explicitly to `BedrockModel` |
| Agent loops forever on tool calls | Tool always returns an error; model retries indefinitely | Ensure tool returns a meaningful string even on failure; add `max_iterations` guard |
| Agent ignores tools; never calls them | Tool schema invalid or docstring missing | Add `Args:` / `Returns:` docstring; verify `@tool` decorator is applied |
| Empty `str(result)` | Model returned zero output tokens | Check `max_tokens` setting; verify prompt is not empty |
| Context length exceeded | Conversation history grew too large | Lower `window_size` on `SlidingWindowConversationManager` or call `agent.messages.clear()` |
| Slow first invocation | Cold-start on Bedrock provisioned model | Expected; subsequent calls are faster; consider provisioned throughput |
| `RuntimeError: no running event loop` | Calling `asyncio.run()` inside existing event loop (e.g. Jupyter) | Use `await agent.invoke_async(...)` directly; or `nest_asyncio.apply()` in Jupyter |
| Tool result not reflected in answer | Tool returned `None` | Ensure every tool returns a non-None string or serialisable value |
