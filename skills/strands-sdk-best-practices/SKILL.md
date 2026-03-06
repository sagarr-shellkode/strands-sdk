---
name: strands-sdk-best-practices
description: "Use when applying Strands SDK production patterns, implementing error handling and retries around agent calls, managing API keys securely, selecting the right model for a use case, structuring agent code, or reviewing Strands code for quality issues. Triggers on: best practice, production, error handling, retry, security, model selection, agent structure, async, reuse agent."
---

# Best Practices — Strands SDK

## Overview

Key patterns for production-grade Strands agents: error handling, retries, security, model selection, and code structure.

## Error Handling

```python
from strands import Agent
from strands.models import BedrockModel
from botocore.exceptions import ClientError
import logging

logger = logging.getLogger(__name__)

async def safe_invoke(agent: Agent, prompt: str, fallback: str = "") -> str:
    try:
        result = await agent.invoke_async(prompt)
        return str(result)
    except ClientError as e:
        code = e.response["Error"]["Code"]
        if code == "ThrottlingException":
            logger.warning("Rate limited — back off and retry")
            raise
        elif code == "AccessDeniedException":
            logger.error("IAM permissions missing for Bedrock")
            raise
        else:
            logger.error(f"AWS error: {code}")
            return fallback
    except Exception as e:
        logger.error(f"Agent invocation failed: {e}", exc_info=True)
        return fallback
```

## Retry with Exponential Backoff

```python
import asyncio
from functools import wraps

def with_retry(max_attempts: int = 3, base_delay: float = 1.0):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    logger.warning(f"Attempt {attempt+1} failed, retrying in {delay}s: {e}")
                    await asyncio.sleep(delay)
        return wrapper
    return decorator

@with_retry(max_attempts=3)
async def invoke_with_retry(agent: Agent, prompt: str) -> str:
    return str(await agent.invoke_async(prompt))
```

## Security: API Keys

```python
# NEVER hardcode credentials
# BAD:
# model = BedrockModel(aws_access_key_id="AKIA...", aws_secret_access_key="...")

# GOOD: use environment variables or IAM roles
import os
model = BedrockModel(
    model_id="...",
    region_name=os.environ["AWS_REGION"]
    # boto3 picks up AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / IAM role automatically
)
```

## Model Selection Guide

| Use Case | Model | Reason |
|---|---|---|
| Simple Q&A, fast | `claude-haiku-4-5` | Cheapest, lowest latency |
| Complex reasoning | `claude-sonnet-4-5` | Balance of quality and cost |
| Long documents | `claude-sonnet-4-5` | 200k context window |
| Code generation | `claude-sonnet-4-5` or `opus-4` | Best code quality |
| High-volume batch | `claude-haiku-4-5` | Cost-optimized |

## Agent Structure: One Agent Per Responsibility

```python
# BAD: one agent doing everything
master_agent = Agent(model=model, system_prompt="Do research, write code, and send emails.")

# GOOD: specialized agents
researcher = Agent(model=model, system_prompt="You research topics and summarize findings.")
coder = Agent(model=model, system_prompt="You write clean Python code.")
orchestrator = Agent(model=model, tools=[research_tool, code_tool])
```

## Reuse Agents, Clear State Between Users

```python
# Create once at startup, reuse across requests
agent = Agent(model=BedrockModel(...))

async def handle_request(user_prompt: str) -> str:
    agent.conversation_manager.clear()  # isolate between users
    return str(await agent.invoke_async(user_prompt))
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| One agent for all tasks | Split into specialists; use agents-as-tools |
| Blocking `agent(prompt)` in async code | Use `await agent.invoke_async(prompt)` |
| No error handling around agent calls | Wrap in try/except; provide fallback response |
| Re-creating Agent on every request | Create once, reuse; clear conversation state between users |
| Logging disabled in production | Set up structured logging; use hooks for LLM call logs |
