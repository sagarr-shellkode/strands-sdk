---
name: strands-sdk-best-practices
description: "Use when applying Strands SDK production patterns, implementing error handling and retries around agent calls, managing API keys securely, selecting the right model for a use case, structuring agent code, or reviewing Strands code for quality issues. Triggers on: best practice, production, error handling, retry, security, model selection, agent structure, async, reuse agent."
---

# Best Practices — Strands SDK

## Overview

This guide covers production-grade patterns for Strands agents: architecture, error handling, retries, security, observability, testing, deployment, scalability, disaster recovery, and cost optimization.

---

## 1. Error Handling

Always wrap agent invocations in structured try/except blocks. Distinguish between transient (retriable) errors and permanent failures.

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
        elif code == "ModelNotReadyException":
            logger.warning("Model unavailable — consider fallback model")
            raise
        elif code == "ValidationException":
            logger.error("Invalid request parameters — do not retry")
            return fallback
        else:
            logger.error("AWS error: %s", code, exc_info=True)
            return fallback
    except Exception as e:
        logger.error("Agent invocation failed: %s", e, exc_info=True)
        return fallback
```

---

## 2. Retry with Exponential Backoff and Jitter

Pure exponential backoff causes retry storms. Add jitter to spread retries across time. Only retry transient errors.

```python
import asyncio
import random
from functools import wraps
from botocore.exceptions import ClientError

RETRIABLE_CODES = {"ThrottlingException", "ServiceUnavailableException", "ModelNotReadyException"}

def with_retry(max_attempts: int = 3, base_delay: float = 1.0, max_delay: float = 30.0):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except ClientError as e:
                    code = e.response["Error"]["Code"]
                    if code not in RETRIABLE_CODES or attempt == max_attempts - 1:
                        raise
                    delay = min(base_delay * (2 ** attempt), max_delay)
                    delay += random.uniform(0, delay * 0.1)   # 10% jitter
                    logger.warning(
                        "Attempt %d/%d failed (%s), retrying in %.2fs",
                        attempt + 1, max_attempts, code, delay
                    )
                    await asyncio.sleep(delay)
        return wrapper
    return decorator

@with_retry(max_attempts=4, base_delay=0.5)
async def invoke_with_retry(agent: Agent, prompt: str) -> str:
    return str(await agent.invoke_async(prompt))
```

---

## 3. Circuit Breaker Pattern

Prevent cascading failures when a downstream model or service degrades. Open the circuit after repeated failures; probe with a single request during half-open state.

```python
import time
from enum import Enum
from dataclasses import dataclass, field

class CircuitState(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Blocking calls; service is down
    HALF_OPEN = "half_open" # Probing; one trial request allowed

@dataclass
class CircuitBreaker:
    failure_threshold: int = 5
    recovery_timeout: float = 60.0   # seconds before OPEN -> HALF_OPEN
    _failures: int = field(default=0, init=False)
    _state: CircuitState = field(default=CircuitState.CLOSED, init=False)
    _opened_at: float = field(default=0.0, init=False)

    def allow_request(self) -> bool:
        if self._state == CircuitState.CLOSED:
            return True
        if self._state == CircuitState.OPEN:
            if time.monotonic() - self._opened_at >= self.recovery_timeout:
                self._state = CircuitState.HALF_OPEN
                return True
            return False
        # HALF_OPEN: allow exactly one probe
        return True

    def record_success(self):
        self._failures = 0
        self._state = CircuitState.CLOSED

    def record_failure(self):
        self._failures += 1
        if self._failures >= self.failure_threshold:
            self._state = CircuitState.OPEN
            self._opened_at = time.monotonic()
            logger.error("Circuit OPEN after %d failures", self._failures)

# Usage
_breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=60.0)

async def invoke_guarded(agent: Agent, prompt: str) -> str:
    if not _breaker.allow_request():
        raise RuntimeError("Circuit is OPEN — service unavailable")
    try:
        result = str(await agent.invoke_async(prompt))
        _breaker.record_success()
        return result
    except Exception:
        _breaker.record_failure()
        raise
```

---

## 4. Production Architecture: Agent Pool and Request Queue

Creating a new `Agent` per request wastes initialization time. Keep a pool of warm agents and funnel requests through an async queue.

```python
import asyncio
from contextlib import asynccontextmanager
from strands import Agent
from strands.models import BedrockModel

class AgentPool:
    def __init__(self, pool_size: int, model: BedrockModel, system_prompt: str):
        self._pool: asyncio.Queue[Agent] = asyncio.Queue()
        for _ in range(pool_size):
            self._pool.put_nowait(
                Agent(model=model, system_prompt=system_prompt)
            )

    @asynccontextmanager
    async def acquire(self):
        agent = await self._pool.get()
        try:
            agent.conversation_manager.clear()  # isolate per-request state
            yield agent
        finally:
            self._pool.put_nowait(agent)

# Initialize once at application startup
_pool = AgentPool(
    pool_size=10,
    model=BedrockModel(model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0"),
    system_prompt="You are a helpful assistant."
)

async def handle_request(user_prompt: str) -> str:
    async with _pool.acquire() as agent:
        return str(await agent.invoke_async(user_prompt))
```

Request queue with back-pressure:

```python
_request_queue: asyncio.Queue = asyncio.Queue(maxsize=500)

async def enqueue_request(prompt: str) -> asyncio.Future:
    loop = asyncio.get_event_loop()
    future: asyncio.Future = loop.create_future()
    try:
        _request_queue.put_nowait((prompt, future))
    except asyncio.QueueFull:
        future.set_exception(RuntimeError("Request queue full — shed load"))
    return future

async def queue_worker():
    while True:
        prompt, future = await _request_queue.get()
        try:
            result = await handle_request(prompt)
            if not future.done():
                future.set_result(result)
        except Exception as e:
            if not future.done():
                future.set_exception(e)
        finally:
            _request_queue.task_done()
```

---

## 5. Graceful Shutdown

Drain in-flight requests before terminating. Register signal handlers for SIGTERM and SIGINT.

```python
import asyncio
import signal
import sys

_shutdown_event = asyncio.Event()

def _handle_signal(sig):
    logger.info("Received %s — initiating graceful shutdown", sig.name)
    _shutdown_event.set()

async def run_application():
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, _handle_signal, sig)

    workers = [asyncio.create_task(queue_worker()) for _ in range(10)]

    await _shutdown_event.wait()
    logger.info("Shutdown signal received — draining queue (max 30s)")

    try:
        await asyncio.wait_for(_request_queue.join(), timeout=30.0)
    except asyncio.TimeoutError:
        logger.warning("Queue drain timed out — cancelling remaining tasks")

    for w in workers:
        w.cancel()
    await asyncio.gather(*workers, return_exceptions=True)
    logger.info("Shutdown complete")
```

---

## 6. Health Check Endpoint

Expose a `/health` endpoint that verifies agent reachability. Used by load balancers and container orchestrators.

```python
# FastAPI example
from fastapi import FastAPI, Response
import httpx

app = FastAPI()

@app.get("/health")
async def health():
    checks = {}

    # Check agent pool availability
    checks["agent_pool"] = "ok" if not _pool._pool.empty() else "degraded"

    # Check circuit breaker state
    checks["circuit_breaker"] = _breaker._state.value

    # Lightweight Bedrock reachability probe (no LLM call — list models)
    try:
        import boto3
        boto3.client("bedrock", region_name="us-east-1").list_foundation_models(
            byProvider="Anthropic"
        )
        checks["bedrock"] = "ok"
    except Exception as e:
        checks["bedrock"] = f"error: {e}"

    overall = "healthy" if all(v in ("ok",) for v in checks.values()) else "degraded"
    status_code = 200 if overall == "healthy" else 503
    return Response(
        content=str({"status": overall, "checks": checks}),
        status_code=status_code,
        media_type="application/json"
    )
```

---

## 7. Security Checklist

### API Key and Credential Handling

```python
# NEVER hardcode credentials
# BAD:
# model = BedrockModel(aws_access_key_id="AKIA...", aws_secret_access_key="wJalr...")

# GOOD: environment variables (loaded from .env via python-dotenv in dev, secrets manager in prod)
import os
model = BedrockModel(
    model_id=os.environ["BEDROCK_MODEL_ID"],
    region_name=os.environ["AWS_REGION"]
    # boto3 resolves credentials from env, ~/.aws, or IAM role automatically
)
```

### Secrets Manager Integration (Production)

```python
import boto3
import json
from functools import lru_cache

@lru_cache(maxsize=None)
def get_secret(secret_name: str) -> dict:
    client = boto3.client("secretsmanager")
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# Use at startup — not in hot paths
secrets = get_secret("myapp/bedrock-config")
api_key = secrets["SARVAM_TTS_API_KEY"]
```

### Input Sanitization

```python
import re

MAX_PROMPT_LENGTH = 10_000   # tokens ~= chars / 4; set a hard cap
INJECTION_PATTERNS = [
    r"ignore previous instructions",
    r"<\|im_start\|>",
    r"\[INST\]",
]

def sanitize_prompt(user_input: str) -> str:
    if len(user_input) > MAX_PROMPT_LENGTH:
        raise ValueError(f"Input exceeds max length of {MAX_PROMPT_LENGTH} characters")
    for pattern in INJECTION_PATTERNS:
        if re.search(pattern, user_input, re.IGNORECASE):
            raise ValueError("Potentially adversarial input detected")
    # Strip null bytes and control characters
    return user_input.replace("\x00", "").strip()
```

### Output Validation

```python
from pydantic import BaseModel, ValidationError
import json

class AgentOutput(BaseModel):
    answer: str
    confidence: float
    sources: list[str] = []

def parse_agent_output(raw: str) -> AgentOutput:
    try:
        data = json.loads(raw)
        return AgentOutput(**data)
    except (json.JSONDecodeError, ValidationError) as e:
        logger.error("Agent returned invalid output: %s | raw=%r", e, raw[:200])
        raise ValueError("Invalid agent output schema") from e
```

### Least-Privilege IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel"],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-haiku-4-5*",
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-5*"
      ]
    }
  ]
}
```

Never grant `bedrock:*` or `*`. Scope to exact model ARNs.

---

## 8. Configuration Management

Use Pydantic `BaseSettings` to load and validate all configuration at startup. Fail fast on missing required values.

```python
from pydantic_settings import BaseSettings
from pydantic import Field, field_validator

class AppConfig(BaseSettings):
    # AWS / Bedrock
    aws_region: str = Field("us-east-1", env="AWS_REGION")
    bedrock_model_id: str = Field(..., env="BEDROCK_MODEL_ID")

    # Agent pool
    agent_pool_size: int = Field(10, env="AGENT_POOL_SIZE", ge=1, le=100)
    request_queue_max: int = Field(500, env="REQUEST_QUEUE_MAX", ge=1)

    # Retry / timeouts
    max_retry_attempts: int = Field(3, env="MAX_RETRY_ATTEMPTS", ge=1)
    agent_timeout_seconds: float = Field(30.0, env="AGENT_TIMEOUT_SECONDS", gt=0)

    # Feature flags
    enable_ltm: bool = Field(False, env="ENABLE_LTM")
    enable_run_of_show: bool = Field(False, env="ENABLE_RUN_OF_SHOW")

    @field_validator("bedrock_model_id")
    @classmethod
    def validate_model_id(cls, v: str) -> str:
        if not v.startswith(("anthropic.", "us.anthropic.", "global.anthropic.")):
            raise ValueError(f"Unrecognized model ID prefix: {v}")
        return v

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

# Singleton — load once at startup
config = AppConfig()
```

---

## 9. Structured and Async-Safe Logging

Use `structlog` or Python's `logging` with a JSON formatter. Never use `print()` in production. Include request IDs for traceability. Avoid blocking I/O in async hot paths — use `QueueHandler`.

```python
import logging
import logging.handlers
import json
import queue
import contextvars

request_id_var: contextvars.ContextVar[str] = contextvars.ContextVar("request_id", default="-")

class StructuredFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        log_obj = {
            "ts": self.formatTime(record, "%Y-%m-%dT%H:%M:%S"),
            "level": record.levelname,
            "logger": record.name,
            "request_id": request_id_var.get("-"),
            "msg": record.getMessage(),
        }
        if record.exc_info:
            log_obj["exc"] = self.formatException(record.exc_info)
        return json.dumps(log_obj)

def configure_logging(level: str = "INFO"):
    log_queue: queue.Queue = queue.Queue(maxsize=10_000)
    queue_handler = logging.handlers.QueueHandler(log_queue)

    stream_handler = logging.StreamHandler()
    stream_handler.setFormatter(StructuredFormatter())

    listener = logging.handlers.QueueListener(log_queue, stream_handler, respect_handler_level=True)
    listener.start()

    root = logging.getLogger()
    root.setLevel(level)
    root.addHandler(queue_handler)
    return listener  # keep reference; call listener.stop() on shutdown
```

Log agent call metadata via Strands hooks:

```python
from strands import Agent
from strands.hooks import AgentInitializedEvent, AfterModelInvocationEvent

def on_model_invoked(event: AfterModelInvocationEvent):
    logger.info(
        "Model invoked",
        extra={
            "model_id": event.model_id,
            "input_tokens": event.usage.input_tokens,
            "output_tokens": event.usage.output_tokens,
            "latency_ms": event.latency_ms,
        }
    )

agent = Agent(
    model=model,
    hooks={AfterModelInvocationEvent: on_model_invoked}
)
```

---

## 10. Dependency Injection for Testability

Inject models and tools rather than constructing them inside business logic. This allows unit tests to swap real Bedrock calls for mocks.

```python
from typing import Protocol

class LLMModel(Protocol):
    pass  # Structural typing — any object usable as a model

class AgentFactory:
    def __init__(self, model: LLMModel, system_prompt: str):
        self._model = model
        self._system_prompt = system_prompt

    def create(self) -> Agent:
        return Agent(model=self._model, system_prompt=self._system_prompt)

# Production wiring
from strands.models import BedrockModel
prod_factory = AgentFactory(
    model=BedrockModel(model_id=config.bedrock_model_id, region_name=config.aws_region),
    system_prompt="You are a helpful assistant."
)

# Test wiring — inject a mock model; no Bedrock calls
from unittest.mock import MagicMock
mock_model = MagicMock()
mock_model.invoke.return_value = "mocked response"
test_factory = AgentFactory(model=mock_model, system_prompt="You are a test assistant.")
```

---

## 11. Model Selection Guide

| Use Case | Recommended Model | Reason |
|---|---|---|
| Simple Q&A, classification | `claude-haiku-4-5` | Lowest latency and cost |
| Complex reasoning, summarization | `claude-sonnet-4-5` | Quality/cost balance |
| Long documents (>100k tokens) | `claude-sonnet-4-5` | 200k context window |
| Code generation and review | `claude-sonnet-4-5` or `claude-opus-4` | Best code quality |
| High-volume batch processing | `claude-haiku-4-5` | Cost-optimized at scale |
| Real-time conversational UI | `claude-haiku-4-5` | Sub-second responses |
| Agentic multi-step reasoning | `claude-sonnet-4-5` | Tool use accuracy |
| Fallback / degraded mode | `claude-haiku-4-5` | Always-available tier |

Use the model ID with the full prefix matching your Bedrock cross-region inference profile:

```python
# Direct regional
"anthropic.claude-haiku-4-5-20251001-v1:0"
# Cross-region inference profile (recommended for HA)
"us.anthropic.claude-haiku-4-5-20251001-v1:0"
"global.anthropic.claude-haiku-4-5-20251001-v1:0"
```

---

## 12. Agent Structure: One Agent Per Responsibility

```python
# BAD: monolithic agent
master_agent = Agent(model=model, system_prompt="Do research, write code, and send emails.")

# GOOD: specialized agents composed via orchestrator
researcher = Agent(
    model=model,
    system_prompt="You research topics and return structured summaries."
)
coder = Agent(
    model=model,
    system_prompt="You write clean, tested Python code."
)
emailer = Agent(
    model=model,
    system_prompt="You draft professional emails from a brief."
)

# Orchestrator delegates via agents-as-tools
from strands.tools import tool

@tool
async def research_tool(topic: str) -> str:
    return str(await researcher.invoke_async(f"Research: {topic}"))

@tool
async def code_tool(spec: str) -> str:
    return str(await coder.invoke_async(f"Implement: {spec}"))

orchestrator = Agent(
    model=model,
    tools=[research_tool, code_tool],
    system_prompt="You coordinate research and coding tasks."
)
```

---

## 13. Reuse Agents, Isolate State Between Requests

```python
# Create once at application startup
_agent = Agent(model=BedrockModel(model_id=config.bedrock_model_id))

async def handle_request(user_prompt: str) -> str:
    # Always clear before each user request to prevent state leakage
    _agent.conversation_manager.clear()
    return str(await _agent.invoke_async(user_prompt))
```

For concurrent requests, use the Agent Pool pattern from Section 4.

---

## 14. Testing Strategies

### Unit Tests (Mock the Model)

```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_safe_invoke_returns_fallback_on_error():
    mock_agent = AsyncMock()
    mock_agent.invoke_async.side_effect = Exception("network error")

    result = await safe_invoke(mock_agent, "hello", fallback="fallback")
    assert result == "fallback"

@pytest.mark.asyncio
async def test_circuit_breaker_opens_after_threshold():
    breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=60.0)
    for _ in range(3):
        breaker.record_failure()
    assert not breaker.allow_request()
```

### Integration Tests (Real Bedrock, Isolated Prompts)

```python
import pytest
import os

@pytest.mark.integration
@pytest.mark.skipif(not os.getenv("RUN_INTEGRATION"), reason="requires AWS credentials")
async def test_real_agent_returns_string():
    from strands.models import BedrockModel
    model = BedrockModel(
        model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0",
        region_name="us-east-1"
    )
    agent = Agent(model=model)
    result = str(await agent.invoke_async("Reply with the single word: PONG"))
    assert "PONG" in result.upper()
```

### Contract Tests (Output Schema Validation)

```python
@pytest.mark.asyncio
async def test_agent_output_matches_schema():
    mock_agent = AsyncMock()
    mock_agent.invoke_async.return_value = '{"answer": "Paris", "confidence": 0.95, "sources": []}'

    raw = str(await mock_agent.invoke_async("What is the capital of France?"))
    output = parse_agent_output(raw)
    assert output.answer == "Paris"
    assert 0.0 <= output.confidence <= 1.0
```

Run integration tests separately from unit tests using pytest marks:

```bash
# Unit only (fast, no AWS)
pytest -m "not integration"

# Integration (requires AWS creds)
RUN_INTEGRATION=1 pytest -m integration
```

---

## 15. Performance Profiling and Bottleneck Identification

Track latency at each stage of the pipeline.

```python
import time
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class LatencyTrace:
    request_id: str
    queue_wait_ms: float = 0.0
    agent_invoke_ms: float = 0.0
    tts_ms: float = 0.0
    total_ms: float = 0.0

    def log(self):
        logger.info(
            "Latency trace",
            extra={
                "request_id": self.request_id,
                "queue_wait_ms": self.queue_wait_ms,
                "agent_invoke_ms": self.agent_invoke_ms,
                "tts_ms": self.tts_ms,
                "total_ms": self.total_ms,
            }
        )

async def timed_invoke(agent: Agent, prompt: str, trace: LatencyTrace) -> str:
    t0 = time.perf_counter()
    result = str(await agent.invoke_async(prompt))
    trace.agent_invoke_ms = (time.perf_counter() - t0) * 1000
    return result
```

Profile async code with `py-spy` or `austin`:

```bash
# Sample a running Python process
py-spy top --pid <PID>

# Record a flame graph
py-spy record -o profile.svg --pid <PID> --duration 30
```

Common bottlenecks and remediation:

| Bottleneck | Symptom | Fix |
|---|---|---|
| Small agent pool | Queue depth grows | Increase `AGENT_POOL_SIZE` |
| Blocking sync calls in async path | Event loop stalls | Use `asyncio.to_thread()` for sync ops |
| Model cold-start latency | First request slow | Warm models with a no-op call at startup |
| Token budget too large | High latency, high cost | Set `max_tokens` on model; trim context |
| Sequential tool calls | Long chain latency | Parallelize independent tools |

---

## 16. Deployment Patterns

### AWS Lambda (Serverless)

Best for low-traffic or event-driven workloads. Avoid cold starts by provisioning concurrency.

```python
# handler.py
import asyncio
from strands import Agent
from strands.models import BedrockModel

# Module-level initialization — reused across warm invocations
_model = BedrockModel(
    model_id=os.environ["BEDROCK_MODEL_ID"],
    region_name=os.environ["AWS_REGION"]
)
_agent = Agent(model=_model, system_prompt="You are a helpful assistant.")

def lambda_handler(event, context):
    prompt = event.get("prompt", "")
    _agent.conversation_manager.clear()
    result = asyncio.run(_agent.invoke_async(prompt))
    return {"statusCode": 200, "body": str(result)}
```

Lambda configuration:
- Memory: 512 MB minimum; increase for large context windows.
- Timeout: Set to `agent_timeout + 5s` buffer (max 15 minutes).
- Provisioned concurrency: Enable for latency-sensitive endpoints.
- Layers: Package `strands-agents` and `boto3` in a Lambda layer.

### ECS Fargate (Container)

Best for sustained traffic with predictable load. Use Application Auto Scaling.

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

Use `--workers 1` with uvicorn and scale horizontally via ECS desired count. Multiple uvicorn workers + asyncio agent pool do not mix safely.

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strands-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: strands-agent
  template:
    spec:
      containers:
      - name: agent
        image: myrepo/strands-agent:latest
        ports:
        - containerPort: 8000
        env:
        - name: BEDROCK_MODEL_ID
          valueFrom:
            secretKeyRef:
              name: bedrock-config
              key: model_id
        - name: AGENT_POOL_SIZE
          value: "10"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2"
            memory: "2Gi"
```

---

## 17. Scalability Patterns

### Horizontal Scaling

- Stateless application design: store session state in DynamoDB or ElastiCache, not in-process.
- Use an ALB (Application Load Balancer) with sticky sessions disabled unless WebSocket sessions require affinity.
- Target Tracking Auto Scaling on `RequestCountPerTarget` or custom CloudWatch metric (queue depth).

### Load Balancing WebSocket Connections

```python
# Store conversation context externally for cross-instance resumability
import boto3

_ddb = boto3.resource("dynamodb")
_table = _ddb.Table("agent-sessions")

async def save_context(session_id: str, messages: list):
    _table.put_item(Item={"session_id": session_id, "messages": messages})

async def load_context(session_id: str) -> list:
    resp = _table.get_item(Key={"session_id": session_id})
    return resp.get("Item", {}).get("messages", [])
```

### Async Fan-out for Independent Tool Calls

```python
async def run_parallel_tools(agent: Agent, prompts: list[str]) -> list[str]:
    tasks = [agent.invoke_async(p) for p in prompts]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return [
        str(r) if not isinstance(r, Exception) else f"ERROR: {r}"
        for r in results
    ]
```

---

## 18. Disaster Recovery and Degraded Mode

### Fallback Model Chain

When the primary model is unavailable, fall back to a smaller, more available model.

```python
FALLBACK_MODEL_CHAIN = [
    "us.anthropic.claude-sonnet-4-5-20251001-v1:0",
    "us.anthropic.claude-haiku-4-5-20251001-v1:0",
]

async def invoke_with_fallback(prompt: str) -> tuple[str, str]:
    """Returns (result, model_used)."""
    for model_id in FALLBACK_MODEL_CHAIN:
        try:
            model = BedrockModel(model_id=model_id, region_name=config.aws_region)
            agent = Agent(model=model)
            result = str(await agent.invoke_async(prompt))
            if model_id != FALLBACK_MODEL_CHAIN[0]:
                logger.warning("Used fallback model: %s", model_id)
            return result, model_id
        except Exception as e:
            logger.error("Model %s failed: %s", model_id, e)

    raise RuntimeError("All models in fallback chain failed")
```

### Degraded Mode

When all models are unavailable, return a cached or static response to avoid a hard failure.

```python
import functools

@functools.lru_cache(maxsize=256)
def get_cached_response(prompt_hash: str) -> str | None:
    # In production, back this with Redis or DynamoDB
    return None

async def invoke_with_degraded_mode(prompt: str, static_fallback: str) -> str:
    import hashlib
    prompt_hash = hashlib.sha256(prompt.encode()).hexdigest()[:16]

    cached = get_cached_response(prompt_hash)
    if cached:
        logger.info("Serving cached response for hash %s", prompt_hash)
        return cached

    try:
        result, _ = await invoke_with_fallback(prompt)
        return result
    except RuntimeError:
        logger.error("All models unavailable — serving static fallback")
        return static_fallback
```

---

## 19. Cost Optimization Checklist

- [ ] Use `claude-haiku-4-5` for classification, routing, and simple Q&A.
- [ ] Set `max_tokens` on model invocations to cap output length.
- [ ] Implement response caching (Redis/DynamoDB) for repeated or near-duplicate prompts.
- [ ] Use batch inference (Bedrock Batch API) for offline workloads — up to 50% discount.
- [ ] Trim conversation history: keep only the last N turns instead of full history.
- [ ] Use `max_turns` on agents to prevent runaway multi-step loops.
- [ ] Monitor token usage via CloudWatch metrics (`bedrock:InputTokens`, `bedrock:OutputTokens`).
- [ ] Set billing alerts at 80% and 100% of monthly budget.
- [ ] Compress system prompts — remove redundant instructions.
- [ ] Evaluate prompt caching (Anthropic) for system prompts used across many requests.

```python
# Cap conversation history to last N turns
def trim_history(agent: Agent, max_turns: int = 10):
    manager = agent.conversation_manager
    messages = manager.get_messages()
    if len(messages) > max_turns * 2:   # each turn = 1 user + 1 assistant
        manager.set_messages(messages[-(max_turns * 2):])
```

---

## 20. Pre-Launch Production Readiness Checklist

### Infrastructure
- [ ] Health check endpoint (`/health`) returns 200 under normal conditions and 503 under degraded conditions.
- [ ] Readiness and liveness probes configured in the container orchestrator.
- [ ] Auto Scaling configured on at least one metric (CPU, queue depth, request count).
- [ ] Graceful shutdown drains in-flight requests before container termination.
- [ ] Circuit breaker configured with tested thresholds.

### Security
- [ ] No credentials or API keys in source code or Docker images.
- [ ] Secrets loaded from AWS Secrets Manager or Parameter Store.
- [ ] IAM role scoped to minimum required `bedrock:InvokeModel` on specific model ARNs.
- [ ] Input length and content validated before reaching agent.
- [ ] Output schema validated before returning to client.
- [ ] HTTPS enforced on all public endpoints.

### Observability
- [ ] Structured JSON logs emitted to stdout (captured by container runtime).
- [ ] Request IDs propagated through all log entries.
- [ ] Model invocation latency and token counts logged per request.
- [ ] CloudWatch alarms on error rate, p99 latency, and queue depth.
- [ ] Distributed tracing enabled (AWS X-Ray or OpenTelemetry).

### Reliability
- [ ] Retry with exponential backoff and jitter on transient errors.
- [ ] Fallback model chain tested end-to-end.
- [ ] Agent pool size load-tested at 2x expected peak traffic.
- [ ] Request queue back-pressure tested (queue full returns 429, not 500).
- [ ] Timeout set on all agent invocations (`asyncio.wait_for`).

### Cost
- [ ] `max_tokens` set on every model call.
- [ ] Billing alert configured in AWS.
- [ ] Token usage metric tracked per endpoint.
- [ ] Haiku used for all non-reasoning paths.

### Testing
- [ ] Unit tests cover error handling, circuit breaker, and retry logic.
- [ ] Integration tests cover real Bedrock path (run in CI with restricted IAM).
- [ ] Contract tests validate output schema.
- [ ] Load test completed with realistic concurrency profile.

---

## 21. Common Mistakes

| Mistake | Fix |
|---|---|
| One agent for all tasks | Split into specialists; use agents-as-tools |
| Blocking `agent(prompt)` in async code | Use `await agent.invoke_async(prompt)` |
| No error handling around agent calls | Wrap in try/except; provide fallback response |
| Re-creating Agent on every request | Create once, reuse; clear conversation state between users |
| Logging disabled or using print() | Set up structured JSON logging with QueueHandler |
| Hardcoded credentials in source code | Use environment variables or Secrets Manager |
| No circuit breaker on external calls | Add CircuitBreaker; open after N consecutive failures |
| Unlimited prompt length from user input | Validate and cap at MAX_PROMPT_LENGTH |
| No health check endpoint | Expose /health; wire to load balancer and orchestrator probes |
| Full conversation history kept forever | Trim to last N turns; use external session store |
| Ignoring token usage in logs | Log input/output tokens per call; set billing alerts |
| Sequential tool calls that could be parallel | Use asyncio.gather for independent tool invocations |
