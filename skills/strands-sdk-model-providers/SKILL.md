---
name: strands-sdk-model-providers
description: "Use when switching model providers in Strands, using Anthropic API directly instead of Bedrock, configuring LiteLLM for OpenAI or Gemini, creating a custom model provider, or comparing BedrockModel vs AnthropicModel vs LiteLLMModel. Triggers on: AnthropicModel, LiteLLMModel, model provider, custom model, ANTHROPIC_API_KEY, provider config, model parameters."
---

# Model Providers — Strands SDK

## Overview

Strands supports multiple model backends. `BedrockModel` is the default, but you can swap to the Anthropic API directly, LiteLLM (which proxies 100+ providers), or implement a fully custom provider — all without changing your `Agent` code. The model object is passed into `Agent(model=...)` and all provider differences are encapsulated there.

## Quick Reference

| Provider | Class | Install | Best For |
|---|---|---|---|
| AWS Bedrock | `BedrockModel` | built-in | AWS-native, IAM auth, enterprise |
| Anthropic API | `AnthropicModel` | built-in (may need extra) | Direct API access, no AWS dependency |
| LiteLLM | `LiteLLMModel` | `pip install litellm` | Multi-provider, quick switching |
| Custom | Subclass `Model` | built-in | Proprietary APIs, special protocols |

---

## BedrockModel (Default)

`BedrockModel` uses `boto3` under the hood and inherits all AWS credential and region resolution. It supports all Claude models available in Bedrock, as well as cross-region inference profiles.

### Basic Setup

```python
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
    temperature=0.7,
)
```

### Full Parameter Reference

| Parameter | Type | Default | Range / Notes |
|---|---|---|---|
| `model_id` | str | required | Bedrock model ARN or ID string |
| `region_name` | str | `AWS_REGION` env or boto3 default | Any AWS region with Bedrock access |
| `max_tokens` | int | `4096` | 1–200000 depending on model |
| `temperature` | float | `1.0` | 0.0 (deterministic) – 1.0 (most random) |
| `top_p` | float | `1.0` | 0.0–1.0; nucleus sampling cutoff |
| `top_k` | int | `None` | 1–500; limits vocabulary per step |
| `stop_sequences` | list[str] | `[]` | Strings that halt generation immediately |
| `boto3_session` | boto3.Session | `None` | Provide a pre-built session (custom credentials) |
| `boto3_client_config` | botocore.config.Config | `None` | Timeouts, retries, proxy settings |
| `additional_request_fields` | dict | `{}` | Extra fields merged into the Bedrock request body |
| `streaming` | bool | `True` | Set `False` to disable streaming |
| `cache_tools` | str | `None` | `"default"` to enable tool prompt caching |
| `cache_prompt` | str | `None` | `"default"` to enable system prompt caching |

### Commonly Used Model IDs (Bedrock)

| Model | Bedrock ID |
|---|---|
| Claude Haiku 4.5 | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| Claude Sonnet 4.5 | `anthropic.claude-sonnet-4-5-20251001-v1:0` |
| Claude Opus 4 | `anthropic.claude-opus-4-20250514-v1:0` |
| Amazon Nova Micro | `amazon.nova-micro-v1:0` |
| Amazon Nova Lite | `amazon.nova-lite-v1:0` |
| Amazon Nova Pro | `amazon.nova-pro-v1:0` |
| Meta Llama 3.1 70B | `meta.llama3-1-70b-instruct-v1:0` |
| Mistral Large | `mistral.mistral-large-2407-v1:0` |

### Cross-Region Inference (Inference Profiles)

When a model is not available in your primary region, or when you want automatic geographic routing for resilience, use a cross-region inference profile ID instead of the base model ID. Bedrock routes the request to the nearest available region automatically.

```python
from strands.models import BedrockModel

# US cross-region profile — routes across us-east-1, us-west-2, etc.
model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-5-20251001-v1:0",
    region_name="us-east-1",
)

# EU cross-region profile — stays within EU data boundary
model = BedrockModel(
    model_id="eu.anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="eu-west-1",
)

# AP cross-region profile
model = BedrockModel(
    model_id="ap.anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="ap-south-1",
)
```

Cross-region profile ID format: `<geo-prefix>.<base-model-id>`
- Geo prefixes: `us`, `eu`, `ap`
- Base model IDs remain unchanged; only the prefix is prepended.

### Provisioned Throughput

For predictable latency and dedicated capacity, use a Provisioned Throughput ARN instead of the on-demand model ID. You must first create the provisioned throughput in the AWS console or via CLI.

```python
from strands.models import BedrockModel

# Use the Provisioned Throughput ARN as model_id
model = BedrockModel(
    model_id="arn:aws:bedrock:us-east-1:123456789012:provisioned-model/abc123def456",
    region_name="us-east-1",
    max_tokens=8192,
    temperature=0.5,
)
```

Create provisioned throughput via CLI:
```bash
aws bedrock create-provisioned-model-throughput \
  --model-id anthropic.claude-haiku-4-5-20251001-v1:0 \
  --provisioned-model-name my-haiku-pt \
  --model-units 1 \
  --region us-east-1
```

### Custom boto3 Session (Non-Default Credentials)

```python
import boto3
from strands.models import BedrockModel

# Use a specific IAM role or cross-account credentials
session = boto3.Session(
    aws_access_key_id="AKIA...",
    aws_secret_access_key="...",
    region_name="us-east-1",
)

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    boto3_session=session,
)
```

### Timeout and Retry Configuration

```python
import boto3
from botocore.config import Config
from strands.models import BedrockModel

config = Config(
    read_timeout=120,          # seconds before read times out
    connect_timeout=10,        # seconds to establish connection
    retries={"max_attempts": 3, "mode": "adaptive"},
)

model = BedrockModel(
    model_id="anthropic.claude-sonnet-4-5-20251001-v1:0",
    boto3_client_config=config,
)
```

---

## AnthropicModel (Direct API)

Calls the Anthropic API directly (api.anthropic.com) instead of routing through AWS Bedrock. Useful when you want to avoid AWS dependencies, need features available in Anthropic's API before they reach Bedrock, or are not running in AWS.

### Setup

```python
from strands.models.anthropic import AnthropicModel
import os

model = AnthropicModel(
    model_id="claude-haiku-4-5-20251001",
    api_key=os.environ["ANTHROPIC_API_KEY"],
    max_tokens=4096,
    temperature=0.7,
)
```

### Full Parameter Reference

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `model_id` | str | required | Anthropic model string (no `anthropic.` prefix) |
| `api_key` | str | `ANTHROPIC_API_KEY` env | Your Anthropic API key |
| `max_tokens` | int | `4096` | 1–200000 depending on model |
| `temperature` | float | `1.0` | 0.0–1.0 |
| `top_p` | float | `1.0` | 0.0–1.0 |
| `top_k` | int | `None` | 1–500 |
| `stop_sequences` | list[str] | `[]` | Halt generation strings |
| `base_url` | str | `None` | Override API endpoint (e.g. proxy) |
| `timeout` | float | `None` | Request timeout in seconds |
| `max_retries` | int | `2` | Built-in retry count |
| `client` | anthropic.Anthropic | `None` | Provide a pre-built Anthropic client |

### Anthropic API vs Bedrock Model ID Format

| API | Example Model ID |
|---|---|
| Anthropic API | `claude-haiku-4-5-20251001` |
| AWS Bedrock | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| Bedrock cross-region | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |

The Anthropic API uses the shorter form without the `anthropic.` prefix and without the `-v1:0` suffix.

### Using a Proxy or Custom Base URL

```python
from strands.models.anthropic import AnthropicModel

# Route through a local proxy or gateway
model = AnthropicModel(
    model_id="claude-haiku-4-5-20251001",
    api_key="sk-...",
    base_url="https://my-proxy.example.com/v1",
    max_tokens=2048,
)
```

### Package Installation Note

If `from strands.models.anthropic import AnthropicModel` raises an `ImportError`, install the optional dependency:

```bash
pip install strands-agents[anthropic]
# or
pip install anthropic
```

---

## LiteLLMModel (Any Provider)

`LiteLLMModel` wraps the `litellm` library, giving access to 100+ model providers through a single interface. The `model_id` follows LiteLLM's `provider/model-name` format.

### Installation

```bash
pip install litellm
# Provider-specific extras (optional but recommended):
pip install litellm[openai]
pip install litellm[gemini]
pip install litellm[cohere]
```

### Provider Examples (10+)

#### OpenAI

```python
from strands.models.litellm import LiteLLMModel
import os

model = LiteLLMModel(
    model_id="gpt-4o",
    api_key=os.environ["OPENAI_API_KEY"],
    max_tokens=4096,
    temperature=0.7,
)

# GPT-4o Mini (cheaper)
model = LiteLLMModel(
    model_id="gpt-4o-mini",
    api_key=os.environ["OPENAI_API_KEY"],
)
```

#### Google Gemini

```python
model = LiteLLMModel(
    model_id="gemini/gemini-2.0-flash",
    api_key=os.environ["GEMINI_API_KEY"],
    max_tokens=8192,
)

# Gemini Pro
model = LiteLLMModel(
    model_id="gemini/gemini-1.5-pro",
    api_key=os.environ["GEMINI_API_KEY"],
)
```

#### Mistral AI

```python
model = LiteLLMModel(
    model_id="mistral/mistral-large-latest",
    api_key=os.environ["MISTRAL_API_KEY"],
    max_tokens=4096,
)

# Mistral Small (cost-optimized)
model = LiteLLMModel(
    model_id="mistral/mistral-small-latest",
    api_key=os.environ["MISTRAL_API_KEY"],
)
```

#### Cohere

```python
model = LiteLLMModel(
    model_id="cohere/command-r-plus",
    api_key=os.environ["COHERE_API_KEY"],
    max_tokens=4096,
)

model = LiteLLMModel(
    model_id="cohere/command-r",
    api_key=os.environ["COHERE_API_KEY"],
)
```

#### Ollama (Local, No API Key)

```python
# Requires: ollama serve (running locally on port 11434)
model = LiteLLMModel(
    model_id="ollama/llama3.1",
    api_base="http://localhost:11434",
)

model = LiteLLMModel(
    model_id="ollama/mistral",
    api_base="http://localhost:11434",
)

# Remote Ollama instance
model = LiteLLMModel(
    model_id="ollama/llama3.1",
    api_base="http://192.168.1.100:11434",
)
```

#### Azure OpenAI

```python
import os

model = LiteLLMModel(
    model_id="azure/my-gpt4o-deployment",   # deployment name, not model name
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    api_base=os.environ["AZURE_OPENAI_ENDPOINT"],   # https://<resource>.openai.azure.com
    api_version="2024-08-01-preview",
)
```

#### Groq (Fast Inference)

```python
model = LiteLLMModel(
    model_id="groq/llama-3.1-70b-versatile",
    api_key=os.environ["GROQ_API_KEY"],
    max_tokens=8192,
)

model = LiteLLMModel(
    model_id="groq/mixtral-8x7b-32768",
    api_key=os.environ["GROQ_API_KEY"],
)
```

#### Together AI

```python
model = LiteLLMModel(
    model_id="together_ai/meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo",
    api_key=os.environ["TOGETHER_API_KEY"],
)

model = LiteLLMModel(
    model_id="together_ai/mistralai/Mixtral-8x7B-Instruct-v0.1",
    api_key=os.environ["TOGETHER_API_KEY"],
)
```

#### Fireworks AI

```python
model = LiteLLMModel(
    model_id="fireworks_ai/accounts/fireworks/models/llama-v3p1-70b-instruct",
    api_key=os.environ["FIREWORKS_API_KEY"],
)
```

#### Anyscale

```python
model = LiteLLMModel(
    model_id="anyscale/meta-llama/Llama-2-70b-chat-hf",
    api_key=os.environ["ANYSCALE_API_KEY"],
)
```

#### DeepSeek

```python
model = LiteLLMModel(
    model_id="deepseek/deepseek-chat",
    api_key=os.environ["DEEPSEEK_API_KEY"],
)
```

#### Vertex AI (Google Cloud)

```python
# Requires: gcloud auth application-default login
model = LiteLLMModel(
    model_id="vertex_ai/gemini-1.5-pro",
    # Uses Application Default Credentials
)
```

### LiteLLMModel Full Parameter Reference

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `model_id` | str | required | `provider/model-name` format |
| `api_key` | str | env var | Provider-specific env var as fallback |
| `api_base` | str | `None` | Override base URL (Ollama, proxies) |
| `api_version` | str | `None` | Azure API version |
| `max_tokens` | int | `4096` | Max response tokens |
| `temperature` | float | `1.0` | 0.0–2.0 (range varies by provider) |
| `top_p` | float | `None` | Nucleus sampling |
| `stop` | list[str] | `None` | Stop sequences |
| `timeout` | float | `None` | Request timeout in seconds |
| `max_retries` | int | `2` | Built-in retry count |
| `kwargs` | dict | `{}` | Extra fields passed to litellm.completion |

---

## Model Fallback Pattern

When a primary model is unavailable (throttled, region outage, quota exceeded), automatically fall back to a secondary model.

```python
from strands import Agent
from strands.models import BedrockModel
from strands.models.litellm import LiteLLMModel
from botocore.exceptions import ClientError
import logging
import os

logger = logging.getLogger(__name__)

class FallbackModel:
    """Wraps a primary model with a secondary fallback."""

    def __init__(self, primary, fallback):
        self.primary = primary
        self.fallback = fallback
        self._active = primary

    def __getattr__(self, name):
        return getattr(self._active, name)

    def invoke(self, request: dict) -> dict:
        try:
            return self.primary.invoke(request)
        except Exception as e:
            logger.warning(f"Primary model failed ({e}), switching to fallback")
            return self.fallback.invoke(request)

    async def invoke_async(self, request: dict) -> dict:
        try:
            return await self.primary.invoke_async(request)
        except Exception as e:
            logger.warning(f"Primary model failed ({e}), switching to fallback")
            return await self.fallback.invoke_async(request)


# Usage
primary = BedrockModel(
    model_id="anthropic.claude-sonnet-4-5-20251001-v1:0",
    region_name="us-east-1",
)
fallback = LiteLLMModel(
    model_id="gpt-4o-mini",
    api_key=os.environ["OPENAI_API_KEY"],
)

model = FallbackModel(primary=primary, fallback=fallback)
agent = Agent(model=model)
```

### Exception-Based Fallback with Retry

```python
from strands.models import BedrockModel
from strands.models.anthropic import AnthropicModel
from botocore.exceptions import ClientError
import asyncio
import os

primary = BedrockModel(model_id="anthropic.claude-sonnet-4-5-20251001-v1:0")
secondary = AnthropicModel(
    model_id="claude-haiku-4-5-20251001",
    api_key=os.environ["ANTHROPIC_API_KEY"],
)

async def invoke_with_fallback(agent_primary, agent_secondary, prompt: str) -> str:
    try:
        return str(await agent_primary.invoke_async(prompt))
    except ClientError as e:
        code = e.response["Error"]["Code"]
        if code in ("ThrottlingException", "ServiceUnavailableException"):
            logger.warning(f"Bedrock {code} — falling back to Anthropic API")
            return str(await agent_secondary.invoke_async(prompt))
        raise
```

---

## A/B Testing Models (Traffic Splitting)

Route a configurable percentage of traffic to a candidate model to compare quality, cost, or latency before fully switching.

```python
import random
import time
import logging
from strands import Agent
from strands.models import BedrockModel
from strands.models.litellm import LiteLLMModel
from dataclasses import dataclass, field
from typing import Optional
import os

logger = logging.getLogger(__name__)


@dataclass
class ABTestResult:
    model_name: str
    response: str
    latency_ms: float
    prompt: str


class ABTestModel:
    """
    Routes traffic between two models based on a split ratio.
    Logs results for offline comparison.
    """

    def __init__(self, model_a, model_b, model_a_name: str, model_b_name: str,
                 model_b_pct: float = 0.1):
        """
        Args:
            model_a: Control model (receives 1 - model_b_pct of traffic).
            model_b: Candidate model (receives model_b_pct of traffic).
            model_b_pct: Fraction of requests sent to model_b. Default 10%.
        """
        self.model_a = model_a
        self.model_b = model_b
        self.model_a_name = model_a_name
        self.model_b_name = model_b_name
        self.model_b_pct = model_b_pct
        self.results: list[ABTestResult] = []

    def _pick_model(self):
        if random.random() < self.model_b_pct:
            return self.model_b, self.model_b_name
        return self.model_a, self.model_a_name

    def invoke(self, request: dict) -> dict:
        model, name = self._pick_model()
        t0 = time.monotonic()
        response = model.invoke(request)
        latency = (time.monotonic() - t0) * 1000
        self._log(name, request, response, latency)
        return response

    async def invoke_async(self, request: dict) -> dict:
        model, name = self._pick_model()
        t0 = time.monotonic()
        response = await model.invoke_async(request)
        latency = (time.monotonic() - t0) * 1000
        self._log(name, request, response, latency)
        return response

    def _log(self, name, request, response, latency):
        prompt = str(request.get("messages", ""))[:200]
        text = ""
        for block in response.get("content", []):
            if block.get("type") == "text":
                text = block["text"][:200]
                break
        result = ABTestResult(model_name=name, response=text,
                              latency_ms=round(latency, 2), prompt=prompt)
        self.results.append(result)
        logger.info(f"AB test | model={name} latency={latency:.0f}ms")

    def summary(self):
        from collections import defaultdict
        stats = defaultdict(lambda: {"count": 0, "total_ms": 0.0})
        for r in self.results:
            stats[r.model_name]["count"] += 1
            stats[r.model_name]["total_ms"] += r.latency_ms
        for name, s in stats.items():
            avg = s["total_ms"] / s["count"] if s["count"] else 0
            print(f"{name}: {s['count']} requests, avg latency {avg:.0f}ms")


# Usage
control = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0")
candidate = LiteLLMModel(model_id="gpt-4o-mini", api_key=os.environ["OPENAI_API_KEY"])

ab_model = ABTestModel(
    model_a=control,    model_a_name="claude-haiku",
    model_b=candidate,  model_b_name="gpt-4o-mini",
    model_b_pct=0.20,   # send 20% to the candidate
)

agent = Agent(model=ab_model)
agent("Summarize this document...")
ab_model.summary()
```

---

## Custom Model Provider

Subclass `Model` to integrate any API not covered by the built-in providers. The two mandatory methods are `invoke` (sync) and `invoke_async` (async).

### Minimal Custom Model

```python
from strands.models.base import Model
from strands import Agent
import httpx

class MyAPIModel(Model):
    def __init__(self, api_url: str, api_key: str, max_tokens: int = 2048):
        self.api_url = api_url
        self.api_key = api_key
        self.max_tokens = max_tokens

    def invoke(self, request: dict) -> dict:
        messages = request.get("messages", [])
        last_user_msg = next(
            (m["content"] for m in reversed(messages) if m["role"] == "user"), ""
        )
        if isinstance(last_user_msg, list):
            last_user_msg = " ".join(
                b["text"] for b in last_user_msg if b.get("type") == "text"
            )

        resp = httpx.post(
            self.api_url,
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={"prompt": last_user_msg, "max_tokens": self.max_tokens},
            timeout=60,
        )
        resp.raise_for_status()
        text = resp.json()["output"]

        return {
            "role": "assistant",
            "content": [{"type": "text", "text": text}],
            "usage": {"input_tokens": 0, "output_tokens": len(text.split())},
        }

    async def invoke_async(self, request: dict) -> dict:
        # For true async, use httpx.AsyncClient; here we delegate to sync
        return self.invoke(request)


agent = Agent(model=MyAPIModel(api_url="https://my-api.example.com/generate", api_key="..."))
```

### Custom Model with Full Tool Use Support

When you need the model to participate in Strands tool-call cycles (the agent calls tools, results are fed back, the model continues), the `invoke` method must correctly parse and return `tool_use` content blocks.

```python
from strands.models.base import Model
from strands import Agent, tool
import httpx
import json
import uuid


class ToolCapableModel(Model):
    """
    Custom model that supports Strands tool-use protocol.
    The API must accept an OpenAI-compatible tools array and return
    tool_calls in its response.
    """

    def __init__(self, api_url: str, api_key: str, max_tokens: int = 4096):
        self.api_url = api_url
        self.api_key = api_key
        self.max_tokens = max_tokens

    def _build_request_body(self, request: dict) -> dict:
        body = {
            "messages": request.get("messages", []),
            "max_tokens": request.get("max_tokens", self.max_tokens),
            "temperature": request.get("temperature", 0.7),
        }
        if request.get("system"):
            body["system"] = request["system"]
        if request.get("tools"):
            # Convert Strands tool schema to API format
            body["tools"] = [
                {
                    "type": "function",
                    "function": {
                        "name": t["name"],
                        "description": t.get("description", ""),
                        "parameters": t.get("input_schema", {}),
                    },
                }
                for t in request["tools"]
            ]
        return body

    def _parse_response(self, api_resp: dict) -> dict:
        content = []
        finish_reason = api_resp.get("finish_reason", "end_turn")

        # Handle text content
        message = api_resp.get("message", {})
        if message.get("content"):
            content.append({"type": "text", "text": message["content"]})

        # Handle tool calls — convert to Strands tool_use blocks
        for tc in message.get("tool_calls", []):
            content.append({
                "type": "tool_use",
                "id": tc.get("id", str(uuid.uuid4())),
                "name": tc["function"]["name"],
                "input": json.loads(tc["function"].get("arguments", "{}")),
            })

        stop_reason = "tool_use" if finish_reason == "tool_calls" else "end_turn"

        return {
            "role": "assistant",
            "content": content,
            "stop_reason": stop_reason,
            "usage": {
                "input_tokens": api_resp.get("usage", {}).get("prompt_tokens", 0),
                "output_tokens": api_resp.get("usage", {}).get("completion_tokens", 0),
            },
        }

    def invoke(self, request: dict) -> dict:
        body = self._build_request_body(request)
        resp = httpx.post(
            self.api_url,
            headers={
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json",
            },
            json=body,
            timeout=120,
        )
        resp.raise_for_status()
        return self._parse_response(resp.json())

    async def invoke_async(self, request: dict) -> dict:
        body = self._build_request_body(request)
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                self.api_url,
                headers={
                    "Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json",
                },
                json=body,
                timeout=120,
            )
        resp.raise_for_status()
        return self._parse_response(resp.json())


# Attach tools and use the model
@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"The weather in {city} is sunny and 22°C."

model = ToolCapableModel(api_url="https://my-api.example.com/v1/chat", api_key="...")
agent = Agent(model=model, tools=[get_weather])
result = agent("What is the weather in Paris?")
```

---

## Model Selection Factory Pattern

Centralize model creation so the rest of your code never imports provider-specific classes directly.

```python
import os
from strands.models import BedrockModel
from strands.models.anthropic import AnthropicModel
from strands.models.litellm import LiteLLMModel


def create_model(provider: str, **overrides):
    """
    Factory that returns a configured model for the given provider.

    Args:
        provider: One of "bedrock", "anthropic", "openai", "gemini",
                  "mistral", "ollama", "groq", "azure".
        **overrides: Any parameter overrides (e.g., temperature=0.5).

    Returns:
        A Strands-compatible model instance.
    """
    defaults = {"max_tokens": 4096, "temperature": 0.7}
    params = {**defaults, **overrides}

    if provider == "bedrock":
        return BedrockModel(
            model_id=os.environ.get(
                "BEDROCK_MODEL_ID", "anthropic.claude-haiku-4-5-20251001-v1:0"
            ),
            region_name=os.environ.get("AWS_REGION", "us-east-1"),
            **params,
        )
    elif provider == "anthropic":
        return AnthropicModel(
            model_id=os.environ.get("ANTHROPIC_MODEL_ID", "claude-haiku-4-5-20251001"),
            api_key=os.environ["ANTHROPIC_API_KEY"],
            **params,
        )
    elif provider == "openai":
        return LiteLLMModel(
            model_id=os.environ.get("OPENAI_MODEL_ID", "gpt-4o-mini"),
            api_key=os.environ["OPENAI_API_KEY"],
            **params,
        )
    elif provider == "gemini":
        return LiteLLMModel(
            model_id=os.environ.get("GEMINI_MODEL_ID", "gemini/gemini-2.0-flash"),
            api_key=os.environ["GEMINI_API_KEY"],
            **params,
        )
    elif provider == "mistral":
        return LiteLLMModel(
            model_id=os.environ.get("MISTRAL_MODEL_ID", "mistral/mistral-large-latest"),
            api_key=os.environ["MISTRAL_API_KEY"],
            **params,
        )
    elif provider == "ollama":
        return LiteLLMModel(
            model_id=os.environ.get("OLLAMA_MODEL_ID", "ollama/llama3.1"),
            api_base=os.environ.get("OLLAMA_BASE_URL", "http://localhost:11434"),
        )
    elif provider == "groq":
        return LiteLLMModel(
            model_id=os.environ.get("GROQ_MODEL_ID", "groq/llama-3.1-70b-versatile"),
            api_key=os.environ["GROQ_API_KEY"],
            **params,
        )
    elif provider == "azure":
        return LiteLLMModel(
            model_id=f"azure/{os.environ['AZURE_DEPLOYMENT_NAME']}",
            api_key=os.environ["AZURE_OPENAI_API_KEY"],
            api_base=os.environ["AZURE_OPENAI_ENDPOINT"],
            api_version=os.environ.get("AZURE_API_VERSION", "2024-08-01-preview"),
            **params,
        )
    else:
        raise ValueError(f"Unknown provider: {provider!r}. "
                         f"Choose from: bedrock, anthropic, openai, gemini, "
                         f"mistral, ollama, groq, azure")


# Usage — switch providers via environment variable
provider = os.environ.get("MODEL_PROVIDER", "bedrock")
model = create_model(provider, temperature=0.5)

from strands import Agent
agent = Agent(model=model)
```

---

## Cost Comparison Table

Approximate prices as of early 2026. Always check the provider's current pricing page for authoritative figures.

| Provider | Model | Input (per 1M tokens) | Output (per 1M tokens) | Notes |
|---|---|---|---|---|
| AWS Bedrock | Claude Haiku 4.5 | $0.80 | $4.00 | On-demand; no volume discount |
| AWS Bedrock | Claude Sonnet 4.5 | $3.00 | $15.00 | On-demand |
| AWS Bedrock | Claude Opus 4 | $15.00 | $75.00 | On-demand |
| AWS Bedrock | Nova Micro | $0.035 | $0.14 | Amazon-native, lowest cost |
| AWS Bedrock | Nova Lite | $0.06 | $0.24 | Amazon-native |
| AWS Bedrock | Nova Pro | $0.80 | $3.20 | Amazon-native |
| Anthropic API | Claude Haiku 4.5 | $0.80 | $4.00 | Same as Bedrock |
| Anthropic API | Claude Sonnet 4.5 | $3.00 | $15.00 | Same as Bedrock |
| OpenAI | GPT-4o | $2.50 | $10.00 | |
| OpenAI | GPT-4o Mini | $0.15 | $0.60 | Very low cost |
| Google | Gemini 2.0 Flash | $0.10 | $0.40 | |
| Google | Gemini 1.5 Pro | $1.25 | $5.00 | >128k context |
| Mistral | Mistral Large | $3.00 | $9.00 | |
| Mistral | Mistral Small | $0.20 | $0.60 | |
| Groq | Llama 3.1 70B | $0.59 | $0.79 | Very low latency |
| Together AI | Llama 3.1 70B Turbo | $0.88 | $0.88 | |
| Ollama | Any | $0.00 | $0.00 | Self-hosted; hardware cost only |

Prompt caching (BedrockModel with `cache_prompt="default"`) reduces input token costs by ~90% for repeated system prompts.

---

## Latency Benchmarks

Approximate time-to-first-token (TTFT) and total latency for a ~500-token response on a standard prompt. Results vary by load, region, and time of day.

| Provider | Model | TTFT (approx) | Total (approx) | Notes |
|---|---|---|---|---|
| Groq | Llama 3.1 70B | ~150ms | ~700ms | Fastest available (custom silicon) |
| OpenAI | GPT-4o Mini | ~300ms | ~1.5s | Fast, consistent |
| Google | Gemini 2.0 Flash | ~400ms | ~2s | |
| Anthropic API | Claude Haiku 4.5 | ~400ms | ~2s | |
| AWS Bedrock | Claude Haiku 4.5 | ~500ms | ~2.5s | Adds AWS networking overhead |
| OpenAI | GPT-4o | ~500ms | ~3s | |
| AWS Bedrock | Claude Sonnet 4.5 | ~800ms | ~5s | |
| Mistral | Mistral Large | ~600ms | ~4s | |
| Together AI | Llama 3.1 70B Turbo | ~250ms | ~1.2s | |
| Ollama (local, GPU) | Llama 3.1 8B | ~100ms | ~1s | Depends heavily on hardware |
| Ollama (local, CPU) | Llama 3.1 8B | ~500ms | ~15s | CPU inference is slow |

Provisioned Throughput on Bedrock reduces P99 latency variance significantly compared to on-demand.

---

## Troubleshooting

### Import and Installation Errors

| Error | Cause | Fix |
|---|---|---|
| `ImportError: AnthropicModel` | `anthropic` package missing | `pip install strands-agents[anthropic]` or `pip install anthropic` |
| `ImportError: LiteLLMModel` | `litellm` package missing | `pip install litellm` |
| `ModuleNotFoundError: litellm.providers.openai` | Missing provider extra | `pip install litellm[openai]` |
| `ImportError: cannot import name 'BedrockModel'` | Outdated strands version | `pip install --upgrade strands-agents` |

### Bedrock-Specific Errors

| Error | Cause | Fix |
|---|---|---|
| `AccessDeniedException` | SSO role has Bedrock Deny, or missing `bedrock:InvokeModel` | Switch to IAM user credentials; verify IAM policy |
| `ValidationException: model identifier` | Wrong `model_id` string or wrong region | Check exact ID in Bedrock console; verify `region_name` |
| `ResourceNotFoundException` | Model not enabled in this region | Enable model access in Bedrock console → Model access |
| `ThrottlingException` | On-demand quota exceeded | Add retry logic; request quota increase; or use Provisioned Throughput |
| `EndpointResolutionError` | `region_name` not set | Set `AWS_REGION` env var or pass `region_name=` explicitly |
| Provisioned ARN `ValidationException` | Model version mismatch | Ensure ARN points to same model version as your `model_id` |

### Anthropic API Errors

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Invalid or missing API key | Check `ANTHROPIC_API_KEY`; no spaces around `=` in `.env` |
| `529 Overloaded` | API capacity | Retry with exponential backoff |
| Model ID `404` | Using Bedrock-format ID | Remove `anthropic.` prefix and `-v1:0` suffix |

### LiteLLM Errors

| Error | Cause | Fix |
|---|---|---|
| `BadRequestError: model not found` | Wrong model ID string | Check LiteLLM docs for exact provider prefix |
| `AuthenticationError` | Wrong or missing API key | Check env var name for the provider |
| `ConnectionError` (Ollama) | Ollama not running | Run `ollama serve` before invoking |
| `azure deployment not found` | Deployment name mismatch | `model_id` must be `azure/<deployment-name>`, not the model name |

### Custom Model Errors

| Error | Cause | Fix |
|---|---|---|
| Tool calls not executed | `invoke` returns no `tool_use` block | Parse `tool_calls` in API response into `{"type": "tool_use", ...}` blocks |
| Infinite tool loop | `stop_reason` always `"end_turn"` | Return `"stop_reason": "tool_use"` when tool calls are present |
| `KeyError: content` | Return dict missing required keys | Always include `role`, `content` (list), and `usage` keys |

### Model ID Format Reference

```
Bedrock on-demand:      anthropic.claude-haiku-4-5-20251001-v1:0
Bedrock cross-region:   us.anthropic.claude-haiku-4-5-20251001-v1:0
Bedrock provisioned:    arn:aws:bedrock:us-east-1:123456789012:provisioned-model/...
Anthropic API:          claude-haiku-4-5-20251001
LiteLLM (OpenAI):       gpt-4o
LiteLLM (Gemini):       gemini/gemini-2.0-flash
LiteLLM (Ollama):       ollama/llama3.1
LiteLLM (Azure):        azure/<deployment-name>
LiteLLM (Groq):         groq/llama-3.1-70b-versatile
LiteLLM (Mistral):      mistral/mistral-large-latest
LiteLLM (Together AI):  together_ai/meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo
```
