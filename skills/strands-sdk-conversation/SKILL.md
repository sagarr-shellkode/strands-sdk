---
name: strands-sdk-conversation
description: "Use when managing multi-turn conversations, persisting chat history across sessions, resetting agent context, using ConversationManager or SlidingWindowConversationManager, or controlling how many turns of history the agent sees. Triggers on: conversation_manager, chat history, multi-turn, reset conversation, context window, SlidingWindowConversationManager."
---

# Conversation Management — Strands SDK

## Overview

Every `Agent` instance maintains a conversation history — an ordered list of messages exchanged between the user and the model. By default this list lives in memory and grows unboundedly. The `ConversationManager` abstraction controls how that list is stored, trimmed, seeded, and shared across sessions. Understanding the message format and the manager interface is essential for building any stateful or multi-user application.

## Quick Reference

| Task | Code |
|---|---|
| Default (in-memory, unbounded) | `Agent(model=model)` |
| Limit to last N messages | `SlidingWindowConversationManager(window_size=N)` |
| Clear all history | `agent.conversation_manager.clear()` |
| Read current history | `agent.conversation_manager.get_messages()` |
| Seed history at construction | `Agent(model=model, messages=[...])` |
| Manually append a message | `agent.conversation_manager.append_message(msg)` |
| Fork from a checkpoint | Deep-copy `get_messages()`, pass to new `Agent` |
| Estimate token budget | ~4 chars per token; count content strings |
| Export history to JSON | `json.dumps(agent.conversation_manager.get_messages())` |
| Import history from JSON | `Agent(model=model, messages=json.loads(data))` |

---

## Message Format

The Strands SDK uses the same message envelope format as the Anthropic Messages API. Every entry in the history list is a dict with two required keys:

```python
{
    "role": "user" | "assistant",
    "content": <str or list-of-blocks>
}
```

### Simple text content

```python
{"role": "user",    "content": "What is the capital of France?"}
{"role": "assistant","content": "The capital of France is Paris."}
```

### Structured content blocks

When the model uses tools, or when you need to embed images, the `content` value is a list of typed blocks instead of a plain string:

```python
# Text block
{"type": "text", "text": "Here is the result."}

# Tool use block — the model requesting a tool call
{
    "type": "tool_use",
    "id": "toolu_01XYZ",
    "name": "get_weather",
    "input": {"city": "Paris"}
}

# Tool result block — your code returning the tool output
{
    "type": "tool_result",
    "tool_use_id": "toolu_01XYZ",
    "content": "Sunny, 22C"
}
```

A full assistant turn that calls a tool followed by a text response looks like:

```python
# Message 1: assistant requests tool
{
    "role": "assistant",
    "content": [
        {"type": "text", "text": "Let me check the weather."},
        {"type": "tool_use", "id": "toolu_01XYZ", "name": "get_weather", "input": {"city": "Paris"}}
    ]
}

# Message 2: user returns tool result (role is "user" for tool results)
{
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": "toolu_01XYZ", "content": "Sunny, 22C"}
    ]
}

# Message 3: assistant final answer
{
    "role": "assistant",
    "content": [{"type": "text", "text": "It is sunny and 22C in Paris."}]
}
```

### Ordering rules

- Messages must strictly alternate `user` / `assistant` roles.
- The first message must have `role: "user"`.
- `tool_result` blocks always go inside a `"user"` role message immediately after the `"assistant"` message that issued the matching `tool_use`.
- System prompts are NOT part of the messages list — pass them via `Agent(system_prompt="...")`.

---

## Default In-Memory Behavior

```python
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1")
agent = Agent(model=model)

agent("My name is Alice.")
agent("What is my name?")  # correctly answers "Alice" — history is preserved
```

After two turns, `agent.conversation_manager.get_messages()` returns:

```python
[
    {"role": "user",     "content": "My name is Alice."},
    {"role": "assistant","content": "Nice to meet you, Alice!"},
    {"role": "user",     "content": "What is my name?"},
    {"role": "assistant","content": "Your name is Alice."},
]
```

---

## Sliding Window (Limit Context Size)

`SlidingWindowConversationManager` keeps only the most recent `window_size` messages. Older messages are silently dropped before each LLM call.

```python
from strands import Agent
from strands.conversation_manager import SlidingWindowConversationManager

manager = SlidingWindowConversationManager(window_size=10)  # keep last 10 messages
agent = Agent(model=model, conversation_manager=manager)
```

Guidelines for choosing `window_size`:
- Each exchange (one user + one assistant turn) = 2 messages.
- A window of 10 = 5 full exchanges of context.
- For long tool-use flows where one exchange can span 3–4 messages, use a larger window (20–30).
- For single-shot stateless tasks, use `window_size=0` (no history beyond the current prompt).

---

## Clear and Reset

```python
# Clear all history mid-session — next call starts fresh
agent.conversation_manager.clear()

agent("New question on a fresh context.")
```

Use `clear()` when:
- Switching user sessions on a shared agent instance.
- A topic change should not carry forward prior context.
- Recovering from a corrupted or looping conversation state.

---

## Manually Injecting Messages

You can inject system context, fake prior turns, or replay a partial conversation by building the message list directly before invoking the agent.

### Seeding at construction

```python
initial_messages = [
    {"role": "user",      "content": "My name is Bob and I am a software engineer."},
    {"role": "assistant", "content": "Got it, Bob. How can I help you today?"},
]

agent = Agent(model=model, messages=initial_messages)
result = agent("Suggest a project idea for me.")
# Agent already knows the user's name and profession
```

### Injecting a system-style context block as a user turn

The Messages API has no `"system"` role in the messages list. Inject additional context as an early `user` / `assistant` exchange:

```python
context_prime = [
    {
        "role": "user",
        "content": (
            "Before we begin: you are helping a medical professional. "
            "Always recommend consulting a licensed physician."
        )
    },
    {
        "role": "assistant",
        "content": "Understood. I will always recommend consulting a licensed physician."
    },
]

agent = Agent(model=model, system_prompt="You are a medical information assistant.", messages=context_prime)
```

### Appending messages to a live conversation

```python
# Inject an assistant turn that the model did not actually generate
agent.conversation_manager.append_message({
    "role": "assistant",
    "content": "I have retrieved the user's account details from the database."
})

# Now the next user prompt continues from this injected context
agent("Based on that, what plan am I on?")
```

---

## Custom ConversationManager

Subclass `ConversationManager` to implement any storage backend or trimming policy.

### Required interface

```python
from strands.conversation_manager import ConversationManager
from typing import List, Dict, Any

class MyConversationManager(ConversationManager):
    def get_messages(self) -> List[Dict[str, Any]]:
        """Return the full ordered message list."""
        ...

    def append_message(self, message: Dict[str, Any]) -> None:
        """Append a single message to the history."""
        ...

    def clear(self) -> None:
        """Delete all messages."""
        ...
```

### Minimal in-memory implementation

```python
class SimpleConversationManager(ConversationManager):
    def __init__(self):
        self._messages: List[Dict[str, Any]] = []

    def get_messages(self):
        return list(self._messages)

    def append_message(self, message):
        self._messages.append(message)

    def clear(self):
        self._messages.clear()
```

---

## Database-Backed Conversation Persistence

### PostgreSQL (psycopg2)

```python
import json
import psycopg2
from strands.conversation_manager import ConversationManager
from typing import List, Dict, Any

class PostgresConversationManager(ConversationManager):
    """Stores conversation history in a PostgreSQL table.

    Schema:
        CREATE TABLE conversation_messages (
            id         SERIAL PRIMARY KEY,
            session_id TEXT NOT NULL,
            position   INTEGER NOT NULL,
            role       TEXT NOT NULL,
            content    JSONB NOT NULL,
            created_at TIMESTAMPTZ DEFAULT NOW()
        );
        CREATE INDEX ON conversation_messages (session_id, position);
    """

    def __init__(self, session_id: str, dsn: str):
        self.session_id = session_id
        self.conn = psycopg2.connect(dsn)

    def get_messages(self) -> List[Dict[str, Any]]:
        with self.conn.cursor() as cur:
            cur.execute(
                "SELECT role, content FROM conversation_messages "
                "WHERE session_id = %s ORDER BY position",
                (self.session_id,)
            )
            return [{"role": row[0], "content": row[1]} for row in cur.fetchall()]

    def append_message(self, message: Dict[str, Any]) -> None:
        with self.conn.cursor() as cur:
            cur.execute(
                "INSERT INTO conversation_messages (session_id, position, role, content) "
                "VALUES (%s, "
                "  (SELECT COALESCE(MAX(position)+1, 0) FROM conversation_messages WHERE session_id = %s), "
                "  %s, %s)",
                (self.session_id, self.session_id, message["role"], json.dumps(message["content"]))
            )
        self.conn.commit()

    def clear(self) -> None:
        with self.conn.cursor() as cur:
            cur.execute("DELETE FROM conversation_messages WHERE session_id = %s", (self.session_id,))
        self.conn.commit()


# Usage
manager = PostgresConversationManager(
    session_id="user-42-session-7",
    dsn="postgresql://user:pass@localhost:5432/mydb"
)
agent = Agent(model=model, conversation_manager=manager)
```

### DynamoDB

```python
import json
import boto3
from strands.conversation_manager import ConversationManager
from typing import List, Dict, Any

class DynamoConversationManager(ConversationManager):
    """Stores conversation history in a DynamoDB table.

    Table schema:
        Partition key: session_id (S)
        Sort key:      position   (N)

    Create with:
        aws dynamodb create-table \
            --table-name ConversationHistory \
            --attribute-definitions AttributeName=session_id,AttributeType=S AttributeName=position,AttributeType=N \
            --key-schema AttributeName=session_id,KeyType=HASH AttributeName=position,KeyType=RANGE \
            --billing-mode PAY_PER_REQUEST
    """

    def __init__(self, session_id: str, table_name: str = "ConversationHistory", region: str = "us-east-1"):
        self.session_id = session_id
        self.table = boto3.resource("dynamodb", region_name=region).Table(table_name)
        self._next_position: int | None = None

    def _load_next_position(self) -> int:
        if self._next_position is not None:
            return self._next_position
        response = self.table.query(
            KeyConditionExpression="session_id = :sid",
            ExpressionAttributeValues={":sid": self.session_id},
            Select="COUNT"
        )
        self._next_position = response["Count"]
        return self._next_position

    def get_messages(self) -> List[Dict[str, Any]]:
        response = self.table.query(
            KeyConditionExpression="session_id = :sid",
            ExpressionAttributeValues={":sid": self.session_id}
        )
        items = sorted(response["Items"], key=lambda x: int(x["position"]))
        return [{"role": item["role"], "content": json.loads(item["content"])} for item in items]

    def append_message(self, message: Dict[str, Any]) -> None:
        pos = self._load_next_position()
        self.table.put_item(Item={
            "session_id": self.session_id,
            "position": pos,
            "role": message["role"],
            "content": json.dumps(message["content"])
        })
        self._next_position = pos + 1

    def clear(self) -> None:
        messages = self.get_messages()
        with self.table.batch_writer() as batch:
            for i in range(len(messages)):
                batch.delete_item(Key={"session_id": self.session_id, "position": i})
        self._next_position = 0


# Usage
manager = DynamoConversationManager(session_id="user-42-session-7")
agent = Agent(model=model, conversation_manager=manager)
```

### Redis (redis-py)

```python
import json
import redis
from strands.conversation_manager import ConversationManager
from typing import List, Dict, Any

class RedisConversationManager(ConversationManager):
    """Stores conversation history in a Redis list.

    Key format: conversation:<session_id>
    Each list element is a JSON-serialized message dict.
    Optional TTL (seconds) evicts idle sessions automatically.
    """

    def __init__(self, session_id: str, redis_url: str = "redis://localhost:6379/0", ttl: int = 3600):
        self.key = f"conversation:{session_id}"
        self.ttl = ttl
        self.r = redis.from_url(redis_url, decode_responses=True)

    def get_messages(self) -> List[Dict[str, Any]]:
        raw = self.r.lrange(self.key, 0, -1)
        return [json.loads(item) for item in raw]

    def append_message(self, message: Dict[str, Any]) -> None:
        self.r.rpush(self.key, json.dumps(message))
        if self.ttl:
            self.r.expire(self.key, self.ttl)

    def clear(self) -> None:
        self.r.delete(self.key)


# Usage
manager = RedisConversationManager(session_id="user-42-session-7", ttl=1800)
agent = Agent(model=model, conversation_manager=manager)
```

---

## Multi-User Session Isolation

Never share a single agent instance (and its conversation state) across users. Two safe patterns:

### Pattern A — One agent per request (stateless)

```python
from strands import Agent
from strands.models import BedrockModel

_model = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1")

async def handle_chat(session_id: str, user_message: str) -> str:
    """Load session history, invoke, persist updated history."""
    manager = RedisConversationManager(session_id=session_id)
    agent = Agent(model=_model, conversation_manager=manager)
    result = await agent.invoke_async(user_message)
    return str(result)
```

The model is reused (no re-auth cost); only the agent wrapper is created per request. The RedisConversationManager automatically loads prior history for that session.

### Pattern B — Shared agent, clear between users (single-user server)

```python
_agent = Agent(model=_model)

async def handle_request(user_id: str, prompt: str) -> str:
    _agent.conversation_manager.clear()          # wipe previous user's context
    return str(await _agent.invoke_async(prompt))
```

This pattern is only safe when requests are serialized (one user at a time). For concurrent servers, use Pattern A.

### Pattern C — Agent pool keyed by session ID

```python
import asyncio
from collections import defaultdict

_agents: dict[str, Agent] = {}
_lock = asyncio.Lock()

async def get_or_create_agent(session_id: str) -> Agent:
    async with _lock:
        if session_id not in _agents:
            _agents[session_id] = Agent(model=_model)
        return _agents[session_id]

async def handle_chat(session_id: str, message: str) -> str:
    agent = await get_or_create_agent(session_id)
    return str(await agent.invoke_async(message))
```

Add LRU eviction or a max-age check to prevent unbounded memory growth in long-running servers.

---

## Conversation Branching (Forking from a Checkpoint)

Take a snapshot of the conversation at any point and branch into parallel threads — useful for exploring multiple response paths or A/B testing different follow-up prompts.

```python
import copy
from strands import Agent

# Shared context established so far
agent = Agent(model=model)
agent("Tell me about three Python web frameworks.")

# Capture checkpoint
checkpoint = copy.deepcopy(agent.conversation_manager.get_messages())

# Branch A — ask for beginner recommendation
agent_a = Agent(model=model, messages=copy.deepcopy(checkpoint))
result_a = agent_a("Which of those is best for a beginner?")

# Branch B — ask for performance recommendation
agent_b = Agent(model=model, messages=copy.deepcopy(checkpoint))
result_b = agent_b("Which of those has the best performance under load?")

# Both branches have the full prior context; neither affects the other
print("Beginner path:", str(result_a))
print("Performance path:", str(result_b))
```

Always `deepcopy` the checkpoint list before passing it to each branch — otherwise mutations in one branch's manager will corrupt the checkpoint.

---

## Summarization to Compress Long Histories

When a conversation grows too long to fit in the context window, summarize older turns into a single synthetic message, then replace the raw history with the summary plus only the most recent messages.

```python
from strands import Agent

def summarize_history(agent: Agent, keep_last_n: int = 6) -> None:
    """Compress all but the last N messages into a single summary turn."""
    messages = agent.conversation_manager.get_messages()
    if len(messages) <= keep_last_n:
        return  # nothing to compress

    old_messages = messages[:-keep_last_n]
    recent_messages = messages[-keep_last_n:]

    # Build a summary prompt from the old messages
    summary_text = "\n".join(
        f"{m['role'].upper()}: {m['content'] if isinstance(m['content'], str) else '[structured content]'}"
        for m in old_messages
    )

    # Use a throwaway agent (no history) to generate the summary
    summarizer = Agent(model=agent.model, system_prompt="Summarize the following conversation concisely.")
    summary_result = summarizer(summary_text)
    summary = str(summary_result)

    # Rebuild history: one synthetic exchange that carries the summary, then recent turns
    compressed = [
        {"role": "user",      "content": f"[Earlier conversation summary]\n{summary}"},
        {"role": "assistant", "content": "Understood. I have the context from our earlier discussion."},
    ] + recent_messages

    agent.conversation_manager.clear()
    for msg in compressed:
        agent.conversation_manager.append_message(msg)


# Usage
agent = Agent(model=model)
for _ in range(20):
    agent("Continue our story about the dragon.")

summarize_history(agent, keep_last_n=4)
# Now history is: [summary exchange] + [last 4 messages]
```

---

## Context Budget Management

Estimate token usage before each call to avoid hitting model context limits.

### Rule-of-thumb estimator

```python
from typing import Any

def estimate_tokens(messages: list[dict[str, Any]], system_prompt: str = "") -> int:
    """Rough token estimate: ~4 characters per token, +3 overhead per message."""
    total_chars = len(system_prompt)
    for msg in messages:
        content = msg.get("content", "")
        if isinstance(content, str):
            total_chars += len(content)
        elif isinstance(content, list):
            for block in content:
                if block.get("type") == "text":
                    total_chars += len(block.get("text", ""))
                elif block.get("type") == "tool_use":
                    total_chars += len(str(block.get("input", "")))
                elif block.get("type") == "tool_result":
                    total_chars += len(str(block.get("content", "")))
        total_chars += 12  # per-message overhead tokens * 4 chars each
    return total_chars // 4


# Context budget guard
MAX_INPUT_TOKENS = 150_000  # Claude's context window

def guard_context(agent: Agent, new_prompt: str) -> None:
    messages = agent.conversation_manager.get_messages()
    estimated = estimate_tokens(messages, new_prompt)
    if estimated > MAX_INPUT_TOKENS * 0.85:
        print(f"Warning: estimated {estimated} tokens, approaching limit. Summarizing.")
        summarize_history(agent, keep_last_n=6)
```

### Model context window reference

| Model | Context window (tokens) |
|---|---|
| claude-haiku-4-5 | 200,000 |
| claude-sonnet-4-5 | 200,000 |
| claude-opus-4 | 200,000 |

For most applications, trigger summarization when estimated usage exceeds 80% of the window.

---

## Conversation Export and Import

### Export to JSON

```python
import json

def export_conversation(agent: Agent, filepath: str) -> None:
    messages = agent.conversation_manager.get_messages()
    with open(filepath, "w") as f:
        json.dump({"version": 1, "messages": messages}, f, indent=2)

export_conversation(agent, "/tmp/session-42.json")
```

### Import from JSON

```python
def import_conversation(agent: Agent, filepath: str) -> None:
    with open(filepath) as f:
        data = json.load(f)
    agent.conversation_manager.clear()
    for msg in data["messages"]:
        agent.conversation_manager.append_message(msg)

# Restore into a new agent
new_agent = Agent(model=model)
import_conversation(new_agent, "/tmp/session-42.json")
```

### Export to JSONL (streaming-friendly)

```python
def export_jsonl(agent: Agent, filepath: str) -> None:
    with open(filepath, "w") as f:
        for msg in agent.conversation_manager.get_messages():
            f.write(json.dumps(msg) + "\n")

def import_jsonl(agent: Agent, filepath: str) -> None:
    agent.conversation_manager.clear()
    with open(filepath) as f:
        for line in f:
            agent.conversation_manager.append_message(json.loads(line.strip()))
```

### Canonical wire format for cross-system transfer

When sending conversation state over an API or storing in a generic document store, use this envelope:

```python
{
    "schema_version": "1.0",
    "session_id": "user-42-session-7",
    "created_at": "2026-03-06T10:00:00Z",
    "model_id": "anthropic.claude-haiku-4-5-20251001-v1:0",
    "system_prompt": "You are a helpful assistant.",
    "messages": [
        {"role": "user",      "content": "Hello"},
        {"role": "assistant", "content": "Hi there!"}
    ]
}
```

---

## Handling tool_use and tool_result Messages in History

When inspecting or replaying history that includes tool calls, the message structure is more complex. Always validate that every `tool_use` block has a matching `tool_result` before replaying.

### Inspecting tool calls in history

```python
def list_tool_calls(agent: Agent) -> list[dict]:
    """Extract all tool_use blocks from assistant messages."""
    calls = []
    for msg in agent.conversation_manager.get_messages():
        if msg["role"] == "assistant" and isinstance(msg["content"], list):
            for block in msg["content"]:
                if block.get("type") == "tool_use":
                    calls.append({
                        "id": block["id"],
                        "name": block["name"],
                        "input": block["input"]
                    })
    return calls
```

### Replaying a conversation with tool results injected manually

```python
# Scenario: replaying a session where a tool returned different data than originally recorded

history = agent.conversation_manager.get_messages()

# Find and replace a specific tool result
for msg in history:
    if msg["role"] == "user" and isinstance(msg["content"], list):
        for block in msg["content"]:
            if block.get("type") == "tool_result" and block["tool_use_id"] == "toolu_01XYZ":
                block["content"] = "Rainy, 8C"  # updated result

# Replay with modified history
replay_agent = Agent(model=model, messages=history)
result = replay_agent("What should I wear today?")
```

### Stripping tool messages for lightweight export

Sometimes you only want the human-readable turns (no tool internals):

```python
def strip_tool_messages(messages: list[dict]) -> list[dict]:
    """Return only text-only turns; skip tool_use/tool_result messages."""
    clean = []
    for msg in messages:
        content = msg["content"]
        if isinstance(content, str):
            clean.append(msg)
        elif isinstance(content, list):
            text_blocks = [b for b in content if b.get("type") == "text"]
            if text_blocks:
                text = " ".join(b["text"] for b in text_blocks)
                clean.append({"role": msg["role"], "content": text})
    return clean
```

---

## Testing Conversation Flows

### Unit test: history accumulates correctly

```python
def test_history_accumulates():
    from unittest.mock import MagicMock, patch

    mock_model = MagicMock()
    mock_model.invoke.side_effect = [
        {"content": [{"type": "text", "text": "Hello Alice!"}], "usage": {"input_tokens": 10, "output_tokens": 5}},
        {"content": [{"type": "text", "text": "Your name is Alice."}], "usage": {"input_tokens": 30, "output_tokens": 8}},
    ]

    agent = Agent(model=mock_model)
    agent("My name is Alice.")
    agent("What is my name?")

    messages = agent.conversation_manager.get_messages()
    assert len(messages) == 4
    assert messages[0]["role"] == "user"
    assert messages[1]["role"] == "assistant"
    assert "Alice" in str(messages[1]["content"])
```

### Unit test: clear() wipes history

```python
def test_clear_wipes_history():
    agent = Agent(model=mock_model)
    agent.conversation_manager.append_message({"role": "user", "content": "Hello"})
    agent.conversation_manager.clear()
    assert agent.conversation_manager.get_messages() == []
```

### Unit test: sliding window evicts old messages

```python
from strands.conversation_manager import SlidingWindowConversationManager

def test_sliding_window_evicts():
    manager = SlidingWindowConversationManager(window_size=4)
    for i in range(6):
        role = "user" if i % 2 == 0 else "assistant"
        manager.append_message({"role": role, "content": f"Message {i}"})

    visible = manager.get_messages()
    assert len(visible) <= 4
    # The most recent messages should be retained
    assert visible[-1]["content"] == "Message 5"
```

### Integration test: seeded context is visible

```python
def test_seeded_context_is_visible():
    seed = [
        {"role": "user",      "content": "My name is Bob."},
        {"role": "assistant", "content": "Hi Bob!"},
    ]
    agent = Agent(model=model, messages=seed)
    # The agent should know "Bob" without being told again
    result = agent("What is my name?")
    assert "Bob" in str(result)
```

### Testing custom ConversationManager

```python
def test_postgres_manager_roundtrip(pg_dsn):
    manager = PostgresConversationManager(session_id="test-session", dsn=pg_dsn)
    manager.clear()

    manager.append_message({"role": "user",      "content": "Hello"})
    manager.append_message({"role": "assistant", "content": "Hi!"})

    msgs = manager.get_messages()
    assert len(msgs) == 2
    assert msgs[0]["role"] == "user"
    assert msgs[1]["content"] == "Hi!"

    manager.clear()
    assert manager.get_messages() == []
```

---

## Common Mistakes and Troubleshooting

| Mistake / Symptom | Cause | Fix |
|---|---|---|
| Context limit error after many turns | Unbounded in-memory history | Use `SlidingWindowConversationManager(window_size=N)` or call `summarize_history()` |
| Agent from previous session bleeds into new session | Shared agent instance not cleared | Call `agent.conversation_manager.clear()` at session start, or use per-session agents |
| History lost between process restarts | In-memory manager is ephemeral | Serialize with `json.dump(get_messages(), ...)` and reload on startup, or use DB-backed manager |
| `ValidationException: first message must be user role` | History starts with an assistant message | Ensure the first element in the messages list has `role: "user"` |
| `ValidationException: roles must alternate` | Two consecutive user or assistant messages | Ensure strict user/assistant/user/assistant ordering; merge or drop adjacent same-role messages |
| `tool_result` not matched to `tool_use` | Mismatched `tool_use_id` | Verify `tool_result.tool_use_id == tool_use.id` for every tool call/result pair |
| Agent ignores injected history | Messages passed after construction | Pass `messages=` to the `Agent` constructor, not to the manager after construction |
| Multi-user contamination | Single agent shared across concurrent requests | Use per-session managers (Pattern A) or an agent pool (Pattern C) |
| Memory grows without bound in long-running server | Agent pool never evicts | Add LRU eviction or TTL-based expiry to the agent cache |
| Forked branch pollutes original conversation | Checkpoint not deep-copied | Use `copy.deepcopy(get_messages())` before branching |
| Summarization loses important facts | Summarizer agent misses details | Increase `keep_last_n`, or summarize incrementally every N turns rather than all at once |
| Tool-use history causes replay errors | Tool IDs are session-unique | When replaying across sessions, re-issue the tool calls rather than replaying raw IDs |

### Diagnostic snippet

```python
# Print current conversation state for debugging
messages = agent.conversation_manager.get_messages()
print(f"Total messages: {len(messages)}")
for i, msg in enumerate(messages):
    content_preview = str(msg["content"])[:80]
    print(f"  [{i}] {msg['role']}: {content_preview}")
estimated = estimate_tokens(messages)
print(f"Estimated tokens: {estimated}")
```
