---
name: strands-sdk-guardrails
description: "Use when adding content filtering to a Strands agent, using AWS Bedrock Guardrails, blocking harmful inputs or outputs, implementing topic restrictions, PII redaction, or handling GuardrailBlockException. Triggers on: guardrail, content filter, safety, GuardrailConfig, BLOCKED, PII, topic denial, word filter."
---

# Guardrails — Strands SDK

## Overview

Guardrails intercept agent inputs and outputs to enforce safety policies. Strands integrates with AWS Bedrock Guardrails to block harmful content, redact PII, restrict topics, and apply word filters.

## Quick Reference

| Feature | What it does |
|---|---|
| Content filters | Block hate speech, violence, sexual content by severity |
| Topic denial | Block off-topic conversations (e.g., competitor mentions) |
| Word filters | Block specific words or phrases |
| PII redaction | Anonymize personal data in inputs/outputs |
| Grounding | Prevent hallucination by checking against source material |

## Bedrock Guardrail Setup

```python
from strands import Agent
from strands.models import BedrockModel

GUARDRAIL_ID = "your-guardrail-id"      # from AWS Bedrock console or boto3
GUARDRAIL_VERSION = "DRAFT"             # or "1", "2", etc.

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    guardrail_config={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": GUARDRAIL_VERSION,
        "trace": "enabled"
    }
)

agent = Agent(model=model, system_prompt="You are a customer service agent.")
```

## Creating a Guardrail via boto3

```python
import boto3

bedrock = boto3.client("bedrock", region_name="us-east-1")

response = bedrock.create_guardrail(
    name="my-agent-guardrail",
    contentPolicyConfig={
        "filtersConfig": [
            {"type": "HATE", "inputStrength": "HIGH", "outputStrength": "HIGH"},
            {"type": "VIOLENCE", "inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
        ]
    },
    topicPolicyConfig={
        "topicsConfig": [
            {
                "name": "competitor-mention",
                "definition": "Any discussion of competitor products",
                "examples": ["Tell me about OpenAI", "Compare to GPT-4"],
                "type": "DENY"
            }
        ]
    },
    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {"type": "EMAIL", "action": "ANONYMIZE"},
            {"type": "PHONE", "action": "ANONYMIZE"},
        ]
    },
    blockedInputMessaging="I cannot process that request.",
    blockedOutputsMessaging="I cannot provide that information."
)

guardrail_id = response["guardrailId"]
```

## Handling Guardrail Blocks

```python
from strands.exceptions import GuardrailBlockException

try:
    result = agent("How do I harm someone?")
except GuardrailBlockException as e:
    print(f"Blocked: {e.blocked_message}")
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Guardrail ID wrong region | Guardrail must be in same region as the model |
| DRAFT version not working in production | Publish a version in Bedrock console; use version number |
| PII still appearing in output | Set action to `ANONYMIZE` not `BLOCK` for redaction |
| Guardrail blocking valid requests | Lower filter strength or refine topic definitions |
