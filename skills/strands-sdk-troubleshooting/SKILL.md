---
name: strands-sdk-troubleshooting
description: Use when debugging Strands SDK errors, fixing AccessDeniedException or ValidationException from Bedrock, resolving ModuleNotFoundError, agent not calling tools, empty responses, ThrottlingException, context limit exceeded, or any unexpected agent behavior. Triggers on: error, exception, not working, debug, AccessDeniedException, ValidationException, ThrottlingException, ModuleNotFoundError, agent broken.
---

# Troubleshooting — Strands SDK

## Diagnostic Checklist

Before debugging specific errors, run through this checklist:

- [ ] `pip show strands-agents` — is the package installed and current?
- [ ] `aws sts get-caller-identity` — are you using an IAM user (not SSO)?
- [ ] Is `AWS_REGION` set correctly?
- [ ] Is the `model_id` spelled exactly as in the Bedrock console?
- [ ] Does your IAM user have `bedrock:InvokeModel` permission?

## Error Reference

### `ModuleNotFoundError: No module named 'strands'`

```bash
pip install strands-agents
# If using a venv:
/path/to/venv/bin/pip install strands-agents
```

### `AccessDeniedException: User is not authorized to perform bedrock:InvokeModel`

Cause: SSO role has a Bedrock Deny policy, or IAM user is missing permissions.

```bash
# 1. Unset SSO credentials
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE

# 2. Set IAM user credentials
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_REGION=us-east-1

# 3. Verify identity
aws sts get-caller-identity
# Must show: arn:aws:iam::ACCOUNT:user/USERNAME (not a role)
```

IAM policy needed:
```json
{"Effect": "Allow", "Action": "bedrock:InvokeModel", "Resource": "arn:aws:bedrock:REGION::foundation-model/MODEL_ID"}
```

### `ValidationException: The model identifier provided is invalid`

```python
# Bedrock model IDs use this format:
# "anthropic.claude-haiku-4-5-20251001-v1:0"
# NOT "claude-haiku-4-5-20251001" (that is the Anthropic API format)

import boto3
client = boto3.client("bedrock", region_name="us-east-1")
models = client.list_foundation_models()
print([m["modelId"] for m in models["modelSummaries"] if "claude" in m["modelId"]])
```

### `ThrottlingException: Too many requests`

```python
# Add exponential backoff (see strands-sdk-best-practices)
# Quick fix:
import time
time.sleep(2 ** attempt)

# For swarms, cap concurrency (see strands-sdk-agent-swarms):
semaphore = asyncio.Semaphore(3)
```

### Agent Invoked But Never Calls Tools

```python
# 1. Check tool is registered
print([t.__name__ for t in agent.tools])

# 2. Verify tool has a docstring
# 3. Use explicit prompt: f"Use the {tool_name} tool to {task}"
# 4. Check system prompt does not restrict tool use
```

### Empty or Truncated Response

```python
# Increase max_tokens
model = BedrockModel(model_id="...", max_tokens=8192)

# Check stop_sequences are not triggering early
model = BedrockModel(model_id="...", stop_sequences=[])
```

### `RuntimeError: no running event loop`

```python
# In scripts:
import asyncio
asyncio.run(agent.invoke_async("prompt"))

# In FastAPI or existing async context:
result = await agent.invoke_async("prompt")  # NOT agent("prompt")
```

### Context Limit Exceeded

```python
from strands.conversation_manager import SlidingWindowConversationManager
agent = Agent(
    model=model,
    conversation_manager=SlidingWindowConversationManager(window_size=10)
)
```

## Debug Logging

```python
import logging
logging.basicConfig(level=logging.DEBUG)
logging.getLogger("strands").setLevel(logging.DEBUG)
logging.getLogger("botocore").setLevel(logging.INFO)  # reduce boto3 noise
```

## Common Mistakes Quick-Reference

| Symptom | Most Likely Cause |
|---|---|
| `str(result)` shows object repr | Use `result.message` or `str(result).strip()` |
| Agent loops without stopping | Check for infinite tool-call loop; add max_turns |
| Tool called with wrong args | Improve docstring parameter descriptions |
| Full token cost despite caching | Content block is under 1024 tokens |
| Agent gives wrong answer repeatedly | Conversation history contaminated — call `clear()` |
