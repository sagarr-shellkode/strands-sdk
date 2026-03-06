---
name: strands-sdk-conversation
description: Use when managing multi-turn conversations, persisting chat history across sessions, resetting agent context, using ConversationManager or SlidingWindowConversationManager, or controlling how many turns of history the agent sees. Triggers on: conversation_manager, chat history, multi-turn, reset conversation, context window, SlidingWindowConversationManager.
---

# Conversation Management — Strands SDK

## Overview

By default, each `Agent` maintains in-memory conversation history. Use `ConversationManager` to control history size, persist across sessions, or clear context between tasks.

## Quick Reference

| Task | Code |
|---|---|
| Default (in-memory) | `Agent(model=model)` — history auto-managed |
| Limit history window | `SlidingWindowConversationManager(window_size=10)` |
| Clear history | `agent.conversation_manager.clear()` |
| Get current history | `agent.conversation_manager.get_messages()` |
| Seed with messages | Pass `messages=[...]` to Agent constructor |

## Default Behavior

```python
from strands import Agent
from strands.models import BedrockModel

agent = Agent(model=BedrockModel(...))

agent("My name is Alice.")
agent("What is my name?")  # correctly answers "Alice"
```

## Sliding Window (Limit Context)

```python
from strands import Agent
from strands.conversation_manager import SlidingWindowConversationManager

manager = SlidingWindowConversationManager(window_size=10)  # keep last 10 messages

agent = Agent(model=model, conversation_manager=manager)
```

## Clear and Reset

```python
# Clear all history mid-session
agent.conversation_manager.clear()

# Start fresh on a new topic
agent("New question on a fresh context.")
```

## Seed with Initial Messages

```python
initial_messages = [
    {"role": "user", "content": "My name is Bob."},
    {"role": "assistant", "content": "Got it, Bob."}
]

agent = Agent(model=model, messages=initial_messages)
result = agent("What is my name?")  # knows the name from seeded context
```

## Persistent Conversation (Custom Store)

```python
import json

# Save history after session
messages = agent.conversation_manager.get_messages()
json.dump(messages, open("history.json", "w"))

# Restore in next session
messages = json.load(open("history.json"))
agent = Agent(model=model, messages=messages)
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Context grows too large, hits token limit | Use `SlidingWindowConversationManager(window_size=N)` |
| Agent remembers previous session data | Call `agent.conversation_manager.clear()` at session start |
| History not saved between restarts | Manually serialize and reload `get_messages()` |
