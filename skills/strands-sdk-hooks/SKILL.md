---
name: strands-sdk-hooks
description: "Use when adding lifecycle hooks to a Strands agent, running code before or after LLM calls, logging agent events, intercepting tool use, implementing custom middleware, or mutating requests before they reach the model. Triggers on: hooks, AgentHook, on_tool_use, on_llm_call, lifecycle, before_invoke, after_invoke, middleware."
---

# Hooks — Strands SDK

## Overview

Hooks intercept agent lifecycle events: before/after LLM calls, before/after tool use, and on errors. Use them for logging, metrics, guardrails, or modifying requests/responses.

## Quick Reference

| Hook Method | When it fires |
|---|---|
| `on_start` | Before each agent invocation |
| `on_llm_call` | Before each LLM API call |
| `on_llm_response` | After each LLM response |
| `on_tool_use` | Before a tool is called |
| `on_tool_result` | After a tool returns a result |
| `on_error` | On any exception |
| `on_end` | After the invocation completes |

## Defining a Hook

```python
from strands import Agent
from strands.hooks import AgentHook
from strands.models import BedrockModel

class LoggingHook(AgentHook):
    def on_llm_call(self, request):
        print(f"[LLM CALL] messages: {len(request.get('messages', []))}")

    def on_llm_response(self, response):
        usage = response.get("usage", {})
        print(f"[LLM RESPONSE] tokens: {usage}")

    def on_tool_use(self, tool_name, tool_input):
        print(f"[TOOL] {tool_name}({tool_input})")

    def on_error(self, error):
        print(f"[ERROR] {type(error).__name__}: {error}")

agent = Agent(
    model=BedrockModel(...),
    hooks=[LoggingHook()]
)
```

## Mutating Requests in Hooks

```python
class SystemPromptInjectorHook(AgentHook):
    def on_llm_call(self, request):
        # Inject extra context into every LLM call
        if request.get("system"):
            request["system"] += "\n\nAlways respond in bullet points."
        return request  # return modified request
```

## Multiple Hooks

```python
agent = Agent(
    model=model,
    hooks=[LoggingHook(), MetricsHook(), SafetyHook()]
)
# Hooks execute in registration order for each event
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Hook method name typo | Verify exact method names from `AgentHook` base class |
| Hook not firing | Ensure hook instance is in the `hooks=[]` list |
| Mutating request but not returning it | Return the modified `request` dict from `on_llm_call` |
| Hook raises exception, breaking agent | Wrap hook logic in `try/except` inside each method |
