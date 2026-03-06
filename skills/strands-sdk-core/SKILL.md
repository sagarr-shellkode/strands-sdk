---
name: strands-sdk-core
description: Use when working with Strands-agents SDK for the first time, importing Agent or BedrockModel, initializing an agent, invoking an agent with a prompt, or asking about basic agent setup. Triggers on: from strands import Agent, BedrockModel, strands-agents, agent initialization, basic invocation.
---

# Core — Strands SDK

## Overview

The Strands-agents SDK lets you build AI agents powered by LLMs. An `Agent` wraps a model and optional tools/system prompt. Call it like a function with a string prompt.

## Quick Reference

| Concept | Import | Example |
|---|---|---|
| Agent | `from strands import Agent` | `agent = Agent(model=model)` |
| BedrockModel | `from strands.models import BedrockModel` | `BedrockModel(model_id="...")` |
| Invoke | — | `result = agent("your prompt")` |
| Async invoke | — | `result = await agent.invoke_async("prompt")` |
| System prompt | constructor | `Agent(model=model, system_prompt="You are...")` |

## Basic Usage

```python
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

agent = Agent(
    model=model,
    system_prompt="You are a helpful assistant."
)

# Synchronous invocation
result = agent("What is the capital of France?")
print(str(result))

# Async invocation
import asyncio
result = await agent.invoke_async("Explain quantum computing.")
```

## Agent Constructor Parameters

| Parameter | Type | Purpose |
|---|---|---|
| `model` | BedrockModel / provider | LLM backend (required) |
| `system_prompt` | str | Agent personality/role |
| `tools` | list | Callable tools the agent can use |
| `conversation_manager` | ConversationManager | Manage multi-turn history |
| `hooks` | list | Lifecycle hooks |
| `guardrail` | Guardrail | Input/output safety layer |
| `trace_attributes` | dict | Observability metadata |

## Common Mistakes

| Mistake | Fix |
|---|---|
| Printing `result` directly gets object repr | Use `str(result)` or `result.message` |
| `agent(prompt)` in async context blocks event loop | Use `await agent.invoke_async(prompt)` |
| Model ID typo causes `ValidationException` | Copy exact ID from Bedrock console |
| Missing `AWS_REGION` env var | Set `region_name` in BedrockModel or export `AWS_REGION` |

## Troubleshooting

- `ModuleNotFoundError: No module named 'strands'` → `pip install strands-agents`
- `ValidationException: model identifier` → check `model_id` spelling and region
- `AccessDeniedException` → IAM user needs `bedrock:InvokeModel` permission; avoid SSO roles
