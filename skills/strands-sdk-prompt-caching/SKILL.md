---
name: strands-sdk-prompt-caching
description: Use when reducing token costs with prompt caching, using cache_point in messages, setting up caching with Bedrock or Anthropic models, checking cache hit metrics, or debugging why caching is not working. Triggers on: cache_point, prompt cache, CacheConfig, reduce cost, cache hit, cache_read_input_tokens.
---

# Prompt Caching — Strands SDK

## Overview

Prompt caching lets you cache frequently-repeated context (system prompt, documents, tool definitions) so re-runs only pay for the uncached portion. Supported on Bedrock Claude models and the Anthropic API.

## Quick Reference

| Concept | Usage |
|---|---|
| Mark cache boundary | `{"cache_point": {"type": "default"}}` in message content |
| System prompt cache | Strands auto-adds cache_point for system prompts |
| Check cache hit | Look for `cache_read_input_tokens` in usage metadata |
| Minimum size | Content before cache_point must be >1024 tokens |
| TTL | 5 minutes (default) |

## Caching the System Prompt

```python
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

# Large system prompt — Strands auto-adds cache_point
system_prompt = "You are an expert..." + " detailed instructions " * 200

agent = Agent(model=model, system_prompt=system_prompt)
```

## Manual cache_point in Messages

```python
messages = [
    {
        "role": "user",
        "content": [
            {"type": "text", "text": "Here is a large document: " + document_text},
            {"cache_point": {"type": "default"}}  # cache everything above this
        ]
    },
    {
        "role": "user",
        "content": "Summarize the key points."  # only this is charged on re-runs
    }
]
```

## Checking Cache Effectiveness

```python
result = agent("Your question")
usage = result.usage
print(usage.get("cache_read_input_tokens", 0))    # tokens served from cache
print(usage.get("cache_creation_input_tokens", 0)) # tokens written to cache
```

## Caching Rules

- Minimum cacheable block: **1024 tokens** (Anthropic requirement)
- Cache TTL: **5 minutes** (varies by provider)
- Cache is per-model, per-account — not shared across users
- Tool definitions are also cacheable — include them before cache_point
- Position matters: everything before cache_point is cached; after is charged

## Common Mistakes

| Mistake | Fix |
|---|---|
| Cache hit never appears | Content before cache_point must be >1024 tokens |
| Cache expires between calls | TTL is 5 min; keep calls within that window |
| Caching on unsupported model | Check Bedrock docs — not all model versions support caching |
| `cache_point` in wrong position | Must be the last item in a content array block |
