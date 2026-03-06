---
name: strands-sdk-model-providers
description: Use when switching model providers in Strands, using Anthropic API directly instead of Bedrock, configuring LiteLLM for OpenAI or Gemini, creating a custom model provider, or comparing BedrockModel vs AnthropicModel vs LiteLLMModel. Triggers on: AnthropicModel, LiteLLMModel, model provider, custom model, ANTHROPIC_API_KEY, provider config, model parameters.
---

# Model Providers — Strands SDK

## Overview

Strands supports multiple model backends. `BedrockModel` is the default, but you can swap to the Anthropic API, LiteLLM (for any provider), or implement a custom provider without changing your Agent code.

## Quick Reference

| Provider | Class | Package |
|---|---|---|
| AWS Bedrock | `BedrockModel` | built-in |
| Anthropic API | `AnthropicModel` | built-in |
| LiteLLM (any) | `LiteLLMModel` | `pip install litellm` |
| Custom | Subclass `Model` | implement `invoke` |

## BedrockModel (Default)

```python
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
    temperature=0.7,
    top_p=0.9
)
```

## AnthropicModel (Direct API)

```python
from strands.models.anthropic import AnthropicModel
import os

model = AnthropicModel(
    model_id="claude-haiku-4-5-20251001",
    api_key=os.environ["ANTHROPIC_API_KEY"],
    max_tokens=4096
)
```

## LiteLLMModel (Any Provider)

```python
from strands.models.litellm import LiteLLMModel
import os

# OpenAI
model = LiteLLMModel(model_id="gpt-4o", api_key=os.environ["OPENAI_API_KEY"])

# Gemini
model = LiteLLMModel(model_id="gemini/gemini-2.0-flash", api_key=os.environ["GEMINI_API_KEY"])

# Ollama (local, no API key needed)
model = LiteLLMModel(model_id="ollama/llama3", api_base="http://localhost:11434")
```

## Custom Model Provider

```python
from strands.models.base import Model

class MyCustomModel(Model):
    def invoke(self, request: dict) -> dict:
        # request contains: messages, system, tools, max_tokens
        response_text = my_api_call(request["messages"])
        return {
            "role": "assistant",
            "content": [{"type": "text", "text": response_text}],
            "usage": {"input_tokens": 0, "output_tokens": 0}
        }

    async def invoke_async(self, request: dict) -> dict:
        return self.invoke(request)

agent = Agent(model=MyCustomModel())
```

## Model Parameters Reference (BedrockModel)

| Parameter | Default | Effect |
|---|---|---|
| `temperature` | 1.0 | Randomness (0=deterministic, 1=creative) |
| `max_tokens` | 4096 | Maximum response length |
| `top_p` | 1.0 | Nucleus sampling threshold |
| `top_k` | None | Top-k sampling |
| `stop_sequences` | [] | Stop generation at these strings |

## Common Mistakes

| Mistake | Fix |
|---|---|
| `AnthropicModel` import fails | Check version; may need `pip install strands-agents[anthropic]` |
| LiteLLM model not found | Install the provider package: `pip install litellm[openai]` |
| Bedrock vs Anthropic model ID format differs | Bedrock: `anthropic.claude-...`; Anthropic API: `claude-...` |
| Custom model does not handle tool calls | Implement tool_use content block parsing in `invoke` return |
