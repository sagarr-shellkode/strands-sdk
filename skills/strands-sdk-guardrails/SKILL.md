---
name: strands-sdk-guardrails
description: "Use when adding content filtering to a Strands agent, using AWS Bedrock Guardrails, blocking harmful inputs or outputs, implementing topic restrictions, PII redaction, or handling GuardrailBlockException. Triggers on: guardrail, content filter, safety, GuardrailConfig, BLOCKED, PII, topic denial, word filter."
---

# Guardrails — Strands SDK

## Overview

Guardrails intercept agent inputs and outputs to enforce safety policies. Strands integrates with AWS Bedrock Guardrails to block harmful content, redact PII, restrict topics, and apply word filters. For cases where Bedrock is unavailable or a lightweight in-process check is needed, hook-based custom guardrails require no external service at all.

---

## Full Bedrock Guardrails Policy Reference

AWS Bedrock Guardrails supports six distinct policy types. All can be combined in a single guardrail.

| Policy Type | What It Does | Key Config Fields |
|---|---|---|
| Content filters | Block hate, insults, sexual content, violence, misconduct, prompt attacks by severity | `type`, `inputStrength`, `outputStrength` |
| Topic denial | Block conversations that stray into defined off-limits topics | `name`, `definition`, `examples`, `type: DENY` |
| Word filters | Block exact words/phrases; use managed lists (profanity) or custom lists | `wordsConfig`, `managedWordListsConfig` |
| PII redaction | Anonymize or block detected personal information entities | `piiEntitiesConfig` with `action: ANONYMIZE` or `BLOCK` |
| Grounding / hallucination check | Compare model output against a reference source; block if not grounded | `groundingThreshold`, `relevanceThreshold` |
| Contextual grounding | For RAG pipelines — validate output faithfulness to retrieved context | `groundingFilter`, `relevanceFilter` |

### Content Filter Strength Values

`NONE`, `LOW`, `MEDIUM`, `HIGH` — apply independently to input and output sides.

```python
"filtersConfig": [
    {"type": "HATE",             "inputStrength": "HIGH",   "outputStrength": "HIGH"},
    {"type": "INSULTS",          "inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
    {"type": "SEXUAL",           "inputStrength": "HIGH",   "outputStrength": "HIGH"},
    {"type": "VIOLENCE",         "inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
    {"type": "MISCONDUCT",       "inputStrength": "LOW",    "outputStrength": "HIGH"},
    {"type": "PROMPT_ATTACK",    "inputStrength": "HIGH",   "outputStrength": "NONE"},
]
```

`PROMPT_ATTACK` only applies to input (jailbreak/injection detection); setting `outputStrength` is ignored but must be provided.

---

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
        "trace": "enabled"              # enables trace data in the response
    }
)

agent = Agent(model=model, system_prompt="You are a customer service agent.")
```

---

## Creating a Guardrail via boto3 (Full Policy Example)

```python
import boto3

bedrock = boto3.client("bedrock", region_name="us-east-1")

response = bedrock.create_guardrail(
    name="my-agent-guardrail",
    description="Production guardrail for customer service agent",

    # 1. Content filters
    contentPolicyConfig={
        "filtersConfig": [
            {"type": "HATE",          "inputStrength": "HIGH",   "outputStrength": "HIGH"},
            {"type": "INSULTS",       "inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
            {"type": "SEXUAL",        "inputStrength": "HIGH",   "outputStrength": "HIGH"},
            {"type": "VIOLENCE",      "inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
            {"type": "MISCONDUCT",    "inputStrength": "LOW",    "outputStrength": "HIGH"},
            {"type": "PROMPT_ATTACK", "inputStrength": "HIGH",   "outputStrength": "NONE"},
        ]
    },

    # 2. Topic denial
    topicPolicyConfig={
        "topicsConfig": [
            {
                "name": "competitor-mention",
                "definition": "Any discussion of competitor products, pricing, or features",
                "examples": [
                    "Tell me about OpenAI",
                    "How does GPT-4 compare?",
                    "Is Gemini better than you?"
                ],
                "type": "DENY"
            },
            {
                "name": "legal-advice",
                "definition": "Providing specific legal advice or interpretation of law",
                "examples": [
                    "Am I liable if my employee gets injured?",
                    "Can I break this contract?"
                ],
                "type": "DENY"
            }
        ]
    },

    # 3. Word filters
    wordPolicyConfig={
        "wordsConfig": [
            {"text": "internal-only"},
            {"text": "confidential project alpha"},
        ],
        "managedWordListsConfig": [
            {"type": "PROFANITY"}   # AWS-managed profanity list
        ]
    },

    # 4. PII redaction
    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {"type": "EMAIL",           "action": "ANONYMIZE"},
            {"type": "PHONE",           "action": "ANONYMIZE"},
            {"type": "NAME",            "action": "ANONYMIZE"},
            {"type": "US_SOCIAL_SECURITY_NUMBER", "action": "BLOCK"},
            {"type": "CREDIT_DEBIT_CARD_NUMBER",  "action": "BLOCK"},
            {"type": "AWS_ACCESS_KEY",  "action": "BLOCK"},
            {"type": "IP_ADDRESS",      "action": "ANONYMIZE"},
            {"type": "ADDRESS",         "action": "ANONYMIZE"},
        ],
        "regexesConfig": [
            {
                "name": "employee-id",
                "description": "Internal employee badge numbers",
                "pattern": r"EMP-\d{6}",
                "action": "ANONYMIZE"
            }
        ]
    },

    # 5. Grounding / hallucination detection
    groundingPolicyConfig={
        "filtersConfig": [
            {
                "type": "GROUNDING",  "threshold": 0.75
            },
            {
                "type": "RELEVANCE",  "threshold": 0.75
            }
        ]
    },

    blockedInputMessaging="I cannot process that request.",
    blockedOutputsMessaging="I cannot provide that information.",

    tags=[
        {"key": "env",     "value": "production"},
        {"key": "project", "value": "customer-service"},
    ]
)

guardrail_id = response["guardrailId"]
print(f"Created guardrail: {guardrail_id}")
```

---

## Grounding / Hallucination Detection Setup

Grounding checks compare the model's response against a reference passage you supply at inference time. It will block responses that are not supported by the source material.

```python
import boto3

bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

# When calling the guardrail directly (outside of Strands):
response = bedrock_runtime.apply_guardrail(
    guardrailIdentifier=GUARDRAIL_ID,
    guardrailVersion=GUARDRAIL_VERSION,
    source="OUTPUT",
    content=[
        {
            "text": {
                "text": "The product was released in 2019.",
                "qualifiers": ["grounding_source"]   # <-- the reference passage
            }
        },
        {
            "text": {
                "text": "The product launched in 2021."  # <-- model output to check
            }
        }
    ]
)

print(response["action"])   # "GUARDRAIL_INTERVENED" if not grounded
```

Within Strands, pass the grounding source as a `qualifiers` block through `BedrockModel`'s `additional_model_request_fields` or include it as a system message segment. Grounding threshold range: `0.0` (block everything) to `1.0` (block nothing); `0.75` is recommended.

---

## Word Filter with Custom Blocklist

```python
import boto3

bedrock = boto3.client("bedrock", region_name="us-east-1")

# Extend an existing guardrail to add a word blocklist
bedrock.update_guardrail(
    guardrailIdentifier=GUARDRAIL_ID,
    name="my-agent-guardrail",
    blockedInputMessaging="I cannot process that request.",
    blockedOutputsMessaging="I cannot provide that information.",
    wordPolicyConfig={
        "wordsConfig": [
            {"text": "acquire"},
            {"text": "merger talks"},
            {"text": "internal roadmap"},
            {"text": "project falcon"},
        ],
        "managedWordListsConfig": [
            {"type": "PROFANITY"}
        ]
    }
)
```

Words are matched case-insensitively. Phrases are matched as substrings, so `"merger talks"` will catch `"our merger talks yesterday"`.

---

## Contextual Grounding Check (RAG Use Case)

In a RAG pipeline the retrieved chunks are the grounding source. Pass them alongside the generated answer so the guardrail can verify the answer is faithful to the retrieved context.

```python
from strands import Agent, tool
from strands.models import BedrockModel
import boto3

bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

@tool
def check_rag_output(retrieved_context: str, generated_answer: str) -> str:
    """Validate that a generated answer is grounded in retrieved context."""
    response = bedrock_runtime.apply_guardrail(
        guardrailIdentifier=GUARDRAIL_ID,
        guardrailVersion=GUARDRAIL_VERSION,
        source="OUTPUT",
        content=[
            {
                "text": {
                    "text": retrieved_context,
                    "qualifiers": ["grounding_source"]
                }
            },
            {
                "text": {"text": generated_answer}
            }
        ]
    )
    if response["action"] == "GUARDRAIL_INTERVENED":
        return "BLOCKED: Answer not grounded in retrieved context."
    return generated_answer


model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    guardrail_config={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": GUARDRAIL_VERSION,
        "trace": "enabled"
    }
)

agent = Agent(model=model, tools=[check_rag_output])
```

---

## Handling Guardrail Blocks

```python
from strands.exceptions import GuardrailBlockException

try:
    result = agent("How do I harm someone?")
except GuardrailBlockException as e:
    print(f"Blocked: {e.blocked_message}")
    # e.blocked_message contains blockedInputMessaging or blockedOutputsMessaging
    # Optionally log e.trace for audit
```

---

## Guardrail Trace Analysis

When `"trace": "enabled"` is set on the guardrail config, Bedrock returns a trace object explaining exactly why content was blocked or modified.

```python
from strands import Agent
from strands.models import BedrockModel
from strands.hooks import AgentHook

class GuardrailTraceHook(AgentHook):
    def on_llm_response(self, response):
        # Bedrock puts guardrail trace inside the raw response metadata
        trace = response.get("amazon-bedrock-guardrailAction", {})
        if not trace:
            return
        action = trace.get("action")          # "BLOCKED" or "NONE"
        if action == "BLOCKED":
            for output in trace.get("outputs", []):
                reason = output.get("text", "")
                print(f"[GUARDRAIL BLOCKED] reason segment: {reason}")
            for assessment in trace.get("assessments", []):
                # Topic policy
                for topic in assessment.get("topicPolicy", {}).get("topics", []):
                    print(f"  Topic denied: {topic['name']} — action: {topic['action']}")
                # Content policy
                for cf in assessment.get("contentPolicy", {}).get("filters", []):
                    print(f"  Content filter: {cf['type']} confidence={cf['confidence']} action={cf['action']}")
                # PII
                for pii in assessment.get("sensitiveInformationPolicy", {}).get("piiEntities", []):
                    print(f"  PII detected: {pii['type']} action={pii['action']}")
                # Word filter
                for w in assessment.get("wordPolicy", {}).get("customWords", []):
                    print(f"  Word blocked: {w['match']}")
                # Grounding
                grounding = assessment.get("groundingPolicy", {})
                if grounding:
                    print(f"  Grounding score: {grounding.get('score')} threshold: {grounding.get('threshold')}")


model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    guardrail_config={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": "DRAFT",
        "trace": "enabled"
    }
)

agent = Agent(model=model, hooks=[GuardrailTraceHook()])
```

---

## Input-Only, Output-Only, and Both-Sides Guardrails

By default, `guardrail_config` applies the guardrail to both the user input and the model output. You can scope it to one side using the `streamProcessingMode` field or by calling `apply_guardrail` directly on only one side.

### Apply only to user input (block bad prompts, never touch model output)

```python
from strands.hooks import AgentHook
import boto3

bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

class InputOnlyGuardrailHook(AgentHook):
    """Block harmful user inputs before they reach the model."""

    def on_llm_call(self, request):
        user_messages = [
            m["content"] for m in request.get("messages", [])
            if m.get("role") == "user"
        ]
        if not user_messages:
            return request
        latest = user_messages[-1]
        text = latest if isinstance(latest, str) else str(latest)

        resp = bedrock_runtime.apply_guardrail(
            guardrailIdentifier=GUARDRAIL_ID,
            guardrailVersion=GUARDRAIL_VERSION,
            source="INPUT",
            content=[{"text": {"text": text}}]
        )
        if resp["action"] == "GUARDRAIL_INTERVENED":
            from strands.exceptions import GuardrailBlockException
            raise GuardrailBlockException("Input blocked by guardrail policy.")
        return request
```

### Apply only to model output (let any prompt in, sanitise the response)

```python
class OutputOnlyGuardrailHook(AgentHook):
    """Redact PII from model output before returning to the caller."""

    def on_llm_response(self, response):
        raw_text = response.get("output", {}).get("message", {}).get("content", [{}])[0].get("text", "")
        if not raw_text:
            return response
        resp = bedrock_runtime.apply_guardrail(
            guardrailIdentifier=GUARDRAIL_ID,
            guardrailVersion=GUARDRAIL_VERSION,
            source="OUTPUT",
            content=[{"text": {"text": raw_text}}]
        )
        if resp["action"] == "GUARDRAIL_INTERVENED":
            # Replace original text with the sanitised version
            sanitised = resp["outputs"][0]["text"]
            response["output"]["message"]["content"][0]["text"] = sanitised
        return response
```

### Both sides via guardrail_config (simplest, recommended default)

```python
model = BedrockModel(
    model_id="...",
    guardrail_config={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": GUARDRAIL_VERSION,
        # no streamProcessingMode needed — applies to both sides by default
    }
)
```

---

## Custom Guardrail Using AgentHook (No Bedrock Required)

When you need lightweight in-process filtering — no AWS dependency, zero latency overhead, and full Python control — implement a guardrail entirely in a hook.

```python
import re
from strands import Agent
from strands.hooks import AgentHook
from strands.models import BedrockModel

# Simple regex-based blocklist
BLOCKED_PATTERNS = [
    re.compile(r"\b(ssn|social security)\b", re.IGNORECASE),
    re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),          # SSN format
    re.compile(r"\b4[0-9]{12}(?:[0-9]{3})?\b"),     # Visa card numbers
]

BLOCKED_TOPICS = ["competitor alpha", "project x", "acquisition target"]


class CustomGuardrailHook(AgentHook):
    """In-process guardrail: no Bedrock call, no latency."""

    def _check_text(self, text: str, source: str) -> None:
        for pattern in BLOCKED_PATTERNS:
            if pattern.search(text):
                raise ValueError(
                    f"[CustomGuardrail] Sensitive pattern detected in {source}: {pattern.pattern}"
                )
        lower = text.lower()
        for topic in BLOCKED_TOPICS:
            if topic in lower:
                raise ValueError(
                    f"[CustomGuardrail] Blocked topic '{topic}' detected in {source}."
                )

    def on_llm_call(self, request):
        for msg in request.get("messages", []):
            content = msg.get("content", "")
            text = content if isinstance(content, str) else str(content)
            self._check_text(text, source="user input")
        return request

    def on_llm_response(self, response):
        text = (
            response.get("output", {})
            .get("message", {})
            .get("content", [{}])[0]
            .get("text", "")
        )
        self._check_text(text, source="model output")
        return response

    def on_error(self, error):
        print(f"[CustomGuardrail] Error caught: {error}")


agent = Agent(
    model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0"),
    hooks=[CustomGuardrailHook()]
)
```

---

## Combining Bedrock Guardrails with Hook-Based Guardrails

Use Bedrock Guardrails for broad policy enforcement (AWS-managed profanity, PII, content filters) and a custom hook for domain-specific logic that Bedrock cannot know about (internal code names, custom regex, business rules).

```python
from strands import Agent
from strands.models import BedrockModel
from strands.hooks import AgentHook
import re

class DomainGuardrailHook(AgentHook):
    """Layer domain-specific rules on top of Bedrock's broad policies."""

    INTERNAL_TERMS = re.compile(
        r"\b(project\s+nighthawk|falcon\s+budget|acquisition\s+pipeline)\b",
        re.IGNORECASE
    )

    def on_llm_call(self, request):
        for msg in request.get("messages", []):
            text = str(msg.get("content", ""))
            if self.INTERNAL_TERMS.search(text):
                raise ValueError("Blocked: internal project term in user input.")
        return request

    def on_llm_response(self, response):
        text = (
            response.get("output", {})
            .get("message", {})
            .get("content", [{}])[0]
            .get("text", "")
        )
        if self.INTERNAL_TERMS.search(text):
            raise ValueError("Blocked: internal project term in model output.")
        return response


# Bedrock Guardrails handles: PII, hate speech, profanity, prompt injection
# DomainGuardrailHook handles: proprietary project names and internal terminology
model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    guardrail_config={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": GUARDRAIL_VERSION,
        "trace": "enabled"
    }
)

agent = Agent(
    model=model,
    hooks=[DomainGuardrailHook()]
)
```

Execution order: the Bedrock guardrail is evaluated by the Bedrock service during the API call itself. Hooks fire around that call — `on_llm_call` fires before the request leaves the process (input-side hook), and `on_llm_response` fires after the response arrives (output-side hook).

---

## Guardrail Versioning and Deployment Workflow

Guardrails follow a DRAFT -> versioned number workflow. DRAFT is mutable; published versions are immutable.

```
DRAFT  ──(create_guardrail_version)──►  "1"  ──►  "2"  ──►  ...
  │                                       │
  │  (edit freely in console or boto3)    │  (immutable — safe for production)
```

```python
import boto3

bedrock = boto3.client("bedrock", region_name="us-east-1")

# 1. Iterate on DRAFT
bedrock.update_guardrail(
    guardrailIdentifier=GUARDRAIL_ID,
    name="my-agent-guardrail",
    blockedInputMessaging="Cannot process.",
    blockedOutputsMessaging="Cannot provide.",
    # ... updated policies ...
)

# 2. When satisfied, publish an immutable version
version_response = bedrock.create_guardrail_version(
    guardrailIdentifier=GUARDRAIL_ID,
    description="Tuned content filters for v2 launch; lowered INSULTS to LOW"
)
published_version = version_response["version"]   # e.g., "2"
print(f"Published version: {published_version}")

# 3. List all versions
versions = bedrock.list_guardrail_versions(guardrailIdentifier=GUARDRAIL_ID)
for v in versions["guardrailVersions"]:
    print(v["version"], v["status"], v["createdAt"])

# 4. Roll back: point the agent at the previous stable version
model = BedrockModel(
    model_id="...",
    guardrail_config={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": "1",   # pinned to old version
    }
)

# 5. Delete an old version when no longer needed
bedrock.delete_guardrail(
    guardrailIdentifier=GUARDRAIL_ID,
    guardrailVersion="1"
)
```

Best practice: always pin production agents to a numbered version. Only development/staging should use `"DRAFT"`.

---

## Testing Guardrails

### Unit-test a specific policy with apply_guardrail

```python
import pytest
import boto3

bedrock_runtime = boto3.client("bedrock-runtime", region_name="us-east-1")

GUARDRAIL_ID = "your-guardrail-id"
GUARDRAIL_VERSION = "DRAFT"


def apply(source: str, text: str) -> dict:
    return bedrock_runtime.apply_guardrail(
        guardrailIdentifier=GUARDRAIL_ID,
        guardrailVersion=GUARDRAIL_VERSION,
        source=source,
        content=[{"text": {"text": text}}]
    )


class TestContentFilter:
    def test_hate_speech_blocked_on_input(self):
        r = apply("INPUT", "I hate all people from that country.")
        assert r["action"] == "GUARDRAIL_INTERVENED"

    def test_benign_input_passes(self):
        r = apply("INPUT", "What are your business hours?")
        assert r["action"] == "NONE"


class TestPIIRedaction:
    def test_email_anonymised_in_output(self):
        r = apply("OUTPUT", "Contact us at alice@example.com for help.")
        assert r["action"] == "GUARDRAIL_INTERVENED"
        sanitised = r["outputs"][0]["text"]
        assert "alice@example.com" not in sanitised
        assert "{EMAIL}" in sanitised or "[redacted]" in sanitised.lower() or "****" in sanitised

    def test_ssn_blocked_in_output(self):
        r = apply("OUTPUT", "Your SSN on file is 123-45-6789.")
        assert r["action"] == "GUARDRAIL_INTERVENED"


class TestTopicDenial:
    def test_competitor_topic_blocked(self):
        r = apply("INPUT", "How does your product compare to OpenAI?")
        assert r["action"] == "GUARDRAIL_INTERVENED"
        assessments = r.get("assessments", [])
        topics = [
            t["name"]
            for a in assessments
            for t in a.get("topicPolicy", {}).get("topics", [])
            if t["action"] == "BLOCKED"
        ]
        assert "competitor-mention" in topics


class TestWordFilter:
    def test_blocked_word_caught(self):
        r = apply("INPUT", "Tell me the internal-only roadmap.")
        assert r["action"] == "GUARDRAIL_INTERVENED"


class TestCustomHookGuardrail:
    """Test the hook-based guardrail without any AWS call."""

    def test_ssn_pattern_raises(self):
        from your_module import CustomGuardrailHook
        hook = CustomGuardrailHook()
        with pytest.raises(ValueError, match="Sensitive pattern"):
            hook._check_text("My SSN is 123-45-6789", source="test")

    def test_clean_text_passes(self):
        from your_module import CustomGuardrailHook
        hook = CustomGuardrailHook()
        hook._check_text("What are your opening hours?", source="test")  # no exception
```

### Integration smoke-test via the agent

```python
from strands import Agent
from strands.models import BedrockModel
from strands.exceptions import GuardrailBlockException

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    guardrail_config={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": GUARDRAIL_VERSION,
    }
)
agent = Agent(model=model)

# Should be blocked
try:
    agent("How do I hurt someone?")
    assert False, "Should have raised GuardrailBlockException"
except GuardrailBlockException:
    pass

# Should pass
result = agent("What is the capital of France?")
assert "Paris" in str(result)
```

---

## Cost of Guardrails: Latency and Token Overhead

### Additional latency

Bedrock Guardrails adds a synchronous evaluation step on every guarded call. Typical overhead:

| Guardrail policies active | Added latency (p50) |
|---|---|
| Content filters only | ~50–100 ms |
| Content + PII + topic | ~100–200 ms |
| Grounding check (large source) | ~200–500 ms |

Latency scales with context length — larger prompts take longer to evaluate.

### Token overhead

Guardrails are billed per text unit processed (input and output separately). As of 2025:
- Content/topic/word/PII policies: charged per 1,000 input or output text characters
- Grounding policy: charged per 1,000 characters of the grounding source in addition to the response

### Minimising cost

```python
# 1. Apply guardrail only to the final user turn, not the full conversation history
#    Use the hook pattern to selectively apply to only the latest message.

# 2. Use DRAFT for development — it incurs the same charges as versioned,
#    but avoids creating unnecessary version snapshots.

# 3. Disable trace in production unless you are actively auditing:
guardrail_config = {
    "guardrailIdentifier": GUARDRAIL_ID,
    "guardrailVersion": "1",
    "trace": "disabled"   # saves trace serialisation overhead
}

# 4. For known-safe internal tools, skip the guardrail entirely by not
#    attaching guardrail_config to the model used by internal sub-agents.
internal_model = BedrockModel(model_id="...")   # no guardrail_config
public_model = BedrockModel(
    model_id="...",
    guardrail_config={"guardrailIdentifier": GUARDRAIL_ID, "guardrailVersion": "1"}
)
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Guardrail ID in wrong region | Guardrail must be in the same AWS region as the Bedrock model call |
| Using `"DRAFT"` in production | Publish an immutable version; pin agents to a version number |
| PII still visible in output | Set action to `"ANONYMIZE"` not `"BLOCK"` — BLOCK replaces with the blocked-output message; ANONYMIZE substitutes a placeholder inline |
| Guardrail blocking valid requests | Lower filter strength or add exempting examples to topic definitions; check trace output to identify which policy fired |
| `PROMPT_ATTACK` filter set on output | `PROMPT_ATTACK` is input-only; Bedrock ignores `outputStrength` but API may reject mismatches |
| Hook not blocking when Bedrock does not | Hook-based guardrails run independently of Bedrock's evaluation; they must be implemented explicitly in `on_llm_call`/`on_llm_response` |
| Grounding check always blocking | Grounding source must be included in the same `apply_guardrail` call; if omitted, the check has no reference and may block everything |
| Version deleted while agents still point to it | Always update all agents to a newer version before deleting an old one |

---

## Expanded Troubleshooting

### `ResourceNotFoundException: Guardrail not found`

**Error code:** `ResourceNotFoundException`

Cause: wrong `guardrailIdentifier`, wrong region, or the guardrail was deleted.

```python
# Verify the guardrail exists in the correct region
bedrock = boto3.client("bedrock", region_name="us-east-1")
try:
    info = bedrock.get_guardrail(
        guardrailIdentifier=GUARDRAIL_ID,
        guardrailVersion="DRAFT"
    )
    print(info["name"], info["status"])
except bedrock.exceptions.ResourceNotFoundException:
    print("Guardrail not found — check ID and region")
```

### `ValidationException: The guardrail version is invalid`

**Error code:** `ValidationException`

Cause: version string `"0"` or a version number that does not exist yet. Version numbers start at `"1"` after the first `create_guardrail_version` call.

```python
# List available versions
resp = bedrock.list_guardrail_versions(guardrailIdentifier=GUARDRAIL_ID)
print([v["version"] for v in resp["guardrailVersions"]])
# Use one of the listed numbers, or "DRAFT"
```

### `AccessDeniedException: not authorized to perform bedrock:ApplyGuardrail`

**Error code:** `AccessDeniedException`

IAM policy fix:

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:ApplyGuardrail"
  ],
  "Resource": [
    "arn:aws:bedrock:REGION::foundation-model/MODEL_ID",
    "arn:aws:bedrock:REGION:ACCOUNT:guardrail/GUARDRAIL_ID"
  ]
}
```

### `ThrottlingException` on guardrail evaluation

Guardrail API calls share the same Bedrock throttle quota as model invocations. Reduce concurrency or request a quota increase in the AWS Service Quotas console.

```python
# Temporary mitigation: exponential backoff on the full agent call
import time

for attempt in range(3):
    try:
        result = agent(prompt)
        break
    except Exception as e:
        if "ThrottlingException" in str(e):
            time.sleep(2 ** attempt)
        else:
            raise
```

### `GuardrailBlockException` raised unexpectedly (false positive)

1. Enable trace: set `"trace": "enabled"` in `guardrail_config`.
2. Add `GuardrailTraceHook` (see Guardrail Trace Analysis section above) to inspect which policy fired and at what confidence.
3. Tune the offending policy: lower the filter strength, add exemption examples to a topic, or remove a word from the blocklist.
4. Test the updated DRAFT with `apply_guardrail` before publishing a new version.

### Hook-based guardrail silently not firing

```python
# Verify the hook is in the agent's hooks list
print(agent.hooks)  # should list your hook instance

# Verify method name is exactly correct
from strands.hooks import AgentHook
import inspect
print([m for m in dir(AgentHook) if not m.startswith("_")])
# on_llm_call, on_llm_response, on_tool_use, on_tool_result, on_error, on_end
```

### Grounding check score always low despite correct context

Cause: grounding source text is too long or unrelated to the specific claim being evaluated. Split into smaller, claim-specific passages rather than dumping the entire document.

```python
# BAD: entire document as grounding source
grounding_source = entire_10000_word_document

# GOOD: only the relevant retrieved chunks
grounding_source = "\n".join(top_3_retrieved_chunks)
```
