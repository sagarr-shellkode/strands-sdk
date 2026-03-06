---
name: strands-sdk-hooks
description: "Use when adding lifecycle hooks to a Strands agent, running code before or after LLM calls, logging agent events, intercepting tool use, implementing custom middleware, or mutating requests before they reach the model. Triggers on: hooks, AgentHook, on_tool_use, on_llm_call, lifecycle, before_invoke, after_invoke, middleware."
---

# Hooks — Strands SDK

## Overview

Hooks intercept agent lifecycle events: before/after LLM calls, before/after tool use, before/after full invocations, and on errors. Use them for logging, metrics, guardrails, caching, rate limiting, retries, or modifying requests/responses in flight.

A hook is a class that inherits from `AgentHook` and overrides one or more event methods. Multiple hooks can be registered on a single agent; they execute in registration order per event. Hook methods that do not return a value (or return `None`) are observation-only. Hook methods that return a value replace the object passed in, enabling request/response mutation.

---

## AgentHook API Reference

### Import

```python
from strands.hooks import AgentHook
```

### Base Class

```python
class AgentHook:
    """Base class for all Strands agent lifecycle hooks.

    Subclass this and override any combination of the methods below.
    Methods you do not override are no-ops by default.
    """

    def on_start(self, invocation_context: dict) -> None: ...
    def on_end(self, invocation_context: dict) -> None: ...

    def on_llm_call(self, request: dict) -> dict | None: ...
    def on_llm_response(self, response: dict) -> dict | None: ...

    def on_tool_use(self, tool_name: str, tool_input: dict) -> dict | None: ...
    def on_tool_result(self, tool_name: str, tool_result: dict) -> dict | None: ...

    def on_error(self, error: Exception) -> None: ...

    # Async variants (override these in async-capable hooks)
    async def on_start_async(self, invocation_context: dict) -> None: ...
    async def on_end_async(self, invocation_context: dict) -> None: ...
    async def on_llm_call_async(self, request: dict) -> dict | None: ...
    async def on_llm_response_async(self, response: dict) -> dict | None: ...
    async def on_tool_use_async(self, tool_name: str, tool_input: dict) -> dict | None: ...
    async def on_tool_result_async(self, tool_name: str, tool_result: dict) -> dict | None: ...
    async def on_error_async(self, error: Exception) -> None: ...
```

### Method Reference

| Method | Fires when | Can mutate? | Sync/Async |
|---|---|---|---|
| `on_start` | Before the agent begins processing a prompt | No (observation only) | Both |
| `on_llm_call` | Before each LLM API call | Yes — return modified `request` dict | Both |
| `on_llm_response` | After each LLM API response is received | Yes — return modified `response` dict | Both |
| `on_tool_use` | Before a tool function is invoked | Yes — return modified `tool_input` dict | Both |
| `on_tool_result` | After a tool function returns | Yes — return modified `tool_result` dict | Both |
| `on_error` | When any exception is raised during invocation | No (observation only) | Both |
| `on_end` | After the full invocation completes (even on error) | No (observation only) | Both |

### Parameter Shapes

**`invocation_context` dict** (passed to `on_start` / `on_end`):
```python
{
    "prompt": str,              # the user prompt string
    "session_id": str,          # unique ID for this invocation
    "agent_id": str,            # agent identifier if set
    "start_time": float,        # unix timestamp (present in on_end)
    "end_time": float,          # unix timestamp (present in on_end only)
    "error": Exception | None,  # set if invocation ended with error (on_end only)
}
```

**`request` dict** (passed to `on_llm_call`):
```python
{
    "model": str,               # model ID string
    "messages": list[dict],     # conversation history in Bedrock format
    "system": list[dict],       # system prompt blocks
    "tools": list[dict],        # tool schemas
    "inferenceConfig": dict,    # max_tokens, temperature, top_p, stop_sequences
}
```

**`response` dict** (passed to `on_llm_response`):
```python
{
    "output": {
        "message": {
            "role": "assistant",
            "content": list[dict],  # text and tool_use content blocks
        }
    },
    "usage": {
        "inputTokens": int,
        "outputTokens": int,
        "totalTokens": int,
        "cacheReadInputTokens": int,    # prompt cache hits
        "cacheWriteInputTokens": int,   # prompt cache writes
    },
    "stopReason": str,          # "end_turn" | "tool_use" | "max_tokens"
    "metrics": {
        "latencyMs": int,       # time for this LLM call in milliseconds
    }
}
```

**`tool_input` dict** (passed to `on_tool_use`): the raw JSON arguments the LLM sent to the tool.

**`tool_result` dict** (passed to `on_tool_result`):
```python
{
    "toolUseId": str,
    "content": list[dict],      # result content blocks
    "status": "success" | "error",
}
```

---

## Hook Execution Order

When multiple hooks are registered, each event fires through all hooks in registration order before the SDK proceeds. The output of hook N becomes the input to hook N+1 for mutating hooks (`on_llm_call`, `on_llm_response`, `on_tool_use`, `on_tool_result`).

```
agent = Agent(model=model, hooks=[HookA(), HookB(), HookC()])

# For on_llm_call:
# 1. request → HookA.on_llm_call  → request_a (or original if None returned)
# 2. request_a → HookB.on_llm_call → request_b
# 3. request_b → HookC.on_llm_call → request_c
# 4. request_c is sent to the LLM

# For on_start / on_end / on_error (observation only):
# All three hooks fire in order; return value is ignored.
```

Full invocation event sequence for a tool-using agent:

```
on_start
  on_llm_call          ← first LLM turn (agent decides to use a tool)
  on_llm_response
    on_tool_use        ← tool is about to be called
    on_tool_result     ← tool returned
  on_llm_call          ← second LLM turn (agent synthesizes tool output)
  on_llm_response
on_end
```

If an exception is raised at any point:

```
on_start
  on_llm_call
  on_error             ← fires with the exception
on_end                 ← always fires, even on error; check context["error"]
```

---

## Defining a Hook

### Minimal example

```python
from strands import Agent
from strands.hooks import AgentHook
from strands.models import BedrockModel

class LoggingHook(AgentHook):
    def on_llm_call(self, request):
        print(f"[LLM CALL] messages: {len(request.get('messages', []))}")

    def on_llm_response(self, response):
        usage = response.get("usage", {})
        print(f"[LLM RESPONSE] tokens in={usage.get('inputTokens')} out={usage.get('outputTokens')}")

    def on_tool_use(self, tool_name, tool_input):
        print(f"[TOOL USE] {tool_name}({tool_input})")

    def on_tool_result(self, tool_name, tool_result):
        status = tool_result.get("status", "unknown")
        print(f"[TOOL RESULT] {tool_name} -> {status}")

    def on_error(self, error):
        print(f"[ERROR] {type(error).__name__}: {error}")

    def on_end(self, invocation_context):
        err = invocation_context.get("error")
        status = "FAILED" if err else "OK"
        print(f"[END] invocation {status}")

agent = Agent(
    model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1"),
    hooks=[LoggingHook()]
)
result = agent("What is 2 + 2?")
```

---

## Async Hook Methods

When `Agent.invoke_async` is used, the SDK calls the `*_async` variants of hook methods if they are defined. If only sync variants exist they are still called. Define async variants when your hook itself does async I/O (e.g., writing to a database or calling an HTTP endpoint).

```python
import aiohttp
from strands.hooks import AgentHook

class RemoteAuditHook(AgentHook):
    def __init__(self, audit_url: str):
        self.audit_url = audit_url

    async def on_llm_call_async(self, request: dict) -> None:
        async with aiohttp.ClientSession() as session:
            await session.post(self.audit_url, json={
                "event": "llm_call",
                "message_count": len(request.get("messages", [])),
            })

    async def on_error_async(self, error: Exception) -> None:
        async with aiohttp.ClientSession() as session:
            await session.post(self.audit_url, json={
                "event": "error",
                "error_type": type(error).__name__,
                "error_message": str(error),
            })
```

Rules for async hooks:
- Override `on_*_async` methods for async behaviour; they are awaited by the SDK.
- If you define both `on_llm_call` and `on_llm_call_async`, the async variant takes precedence when the agent is invoked asynchronously.
- Async hooks that raise exceptions propagate to the caller just like sync hook exceptions.

---

## Request / Response Mutation

### Pattern

Return the (possibly modified) dict from `on_llm_call` or `on_tool_use` to replace the value in-flight. If you return `None` (or omit a return statement) the original object is used unchanged.

```python
class SystemPromptInjectorHook(AgentHook):
    def on_llm_call(self, request: dict) -> dict:
        # Append a runtime instruction to every LLM call
        system_blocks = request.get("system", [])
        for block in system_blocks:
            if block.get("type") == "text":
                block["text"] += "\n\nAlways respond concisely in three sentences or fewer."
        return request
```

### Append to conversation history

```python
class ContextInjectorHook(AgentHook):
    def __init__(self, extra_context: str):
        self.extra_context = extra_context

    def on_llm_call(self, request: dict) -> dict:
        messages = request.get("messages", [])
        if messages:
            # Insert a system-style note before the last user turn
            messages.insert(-1, {
                "role": "user",
                "content": [{"type": "text", "text": f"[Context]: {self.extra_context}"}]
            })
        return request
```

### Modify inference parameters at runtime

```python
class LowTemperatureHook(AgentHook):
    """Force deterministic output for all LLM calls."""

    def on_llm_call(self, request: dict) -> dict:
        cfg = request.setdefault("inferenceConfig", {})
        cfg["temperature"] = 0.0
        cfg["topP"] = 1.0
        return request
```

### Modify tool results before the LLM sees them

```python
class ResultSanitizerHook(AgentHook):
    """Truncate excessively long tool results."""

    MAX_CHARS = 4000

    def on_tool_result(self, tool_name: str, tool_result: dict) -> dict:
        for block in tool_result.get("content", []):
            if block.get("type") == "text":
                text = block["text"]
                if len(text) > self.MAX_CHARS:
                    block["text"] = text[:self.MAX_CHARS] + "\n... [truncated]"
        return tool_result
```

---

## Logging Hook — Structured JSON Output

Emits one JSON line per LLM call and one per tool use, suitable for ingestion into CloudWatch Logs Insights, Datadog, or any structured log pipeline.

```python
import json
import logging
import time
from strands.hooks import AgentHook

logger = logging.getLogger("strands.audit")
logging.basicConfig(
    level=logging.INFO,
    format="%(message)s",   # emit raw JSON lines
)

class StructuredLoggingHook(AgentHook):
    def __init__(self, agent_name: str):
        self.agent_name = agent_name
        self._call_start: float = 0.0

    def on_start(self, invocation_context: dict) -> None:
        logger.info(json.dumps({
            "event": "invocation_start",
            "agent": self.agent_name,
            "session_id": invocation_context.get("session_id"),
            "ts": time.time(),
        }))

    def on_llm_call(self, request: dict) -> None:
        self._call_start = time.time()
        logger.info(json.dumps({
            "event": "llm_call",
            "agent": self.agent_name,
            "model": request.get("model"),
            "message_count": len(request.get("messages", [])),
            "ts": self._call_start,
        }))

    def on_llm_response(self, response: dict) -> None:
        usage = response.get("usage", {})
        latency_ms = int((time.time() - self._call_start) * 1000)
        logger.info(json.dumps({
            "event": "llm_response",
            "agent": self.agent_name,
            "stop_reason": response.get("stopReason"),
            "input_tokens": usage.get("inputTokens", 0),
            "output_tokens": usage.get("outputTokens", 0),
            "cache_read_tokens": usage.get("cacheReadInputTokens", 0),
            "latency_ms": latency_ms,
            "ts": time.time(),
        }))

    def on_tool_use(self, tool_name: str, tool_input: dict) -> None:
        logger.info(json.dumps({
            "event": "tool_use",
            "agent": self.agent_name,
            "tool": tool_name,
            "input_keys": list(tool_input.keys()),
            "ts": time.time(),
        }))

    def on_tool_result(self, tool_name: str, tool_result: dict) -> None:
        logger.info(json.dumps({
            "event": "tool_result",
            "agent": self.agent_name,
            "tool": tool_name,
            "status": tool_result.get("status"),
            "ts": time.time(),
        }))

    def on_error(self, error: Exception) -> None:
        logger.error(json.dumps({
            "event": "error",
            "agent": self.agent_name,
            "error_type": type(error).__name__,
            "error_message": str(error),
            "ts": time.time(),
        }))

    def on_end(self, invocation_context: dict) -> None:
        err = invocation_context.get("error")
        logger.info(json.dumps({
            "event": "invocation_end",
            "agent": self.agent_name,
            "session_id": invocation_context.get("session_id"),
            "status": "error" if err else "success",
            "ts": time.time(),
        }))
```

---

## Metrics Hook — Per-Call Timing

Tracks cumulative token usage and wall-clock time per LLM call. Call `.summary()` after any number of invocations.

```python
import time
from dataclasses import dataclass, field
from typing import List
from strands.hooks import AgentHook

@dataclass
class CallRecord:
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int
    latency_ms: float
    stop_reason: str

class MetricsHook(AgentHook):
    def __init__(self):
        self._call_start: float = 0.0
        self.records: List[CallRecord] = []
        self.tool_call_count: int = 0

    def on_llm_call(self, request: dict) -> None:
        self._call_start = time.perf_counter()

    def on_llm_response(self, response: dict) -> None:
        elapsed_ms = (time.perf_counter() - self._call_start) * 1000
        usage = response.get("usage", {})
        self.records.append(CallRecord(
            input_tokens=usage.get("inputTokens", 0),
            output_tokens=usage.get("outputTokens", 0),
            cache_read_tokens=usage.get("cacheReadInputTokens", 0),
            latency_ms=round(elapsed_ms, 2),
            stop_reason=response.get("stopReason", "unknown"),
        ))

    def on_tool_use(self, tool_name: str, tool_input: dict) -> None:
        self.tool_call_count += 1

    def summary(self) -> dict:
        if not self.records:
            return {"llm_calls": 0}
        total_in = sum(r.input_tokens for r in self.records)
        total_out = sum(r.output_tokens for r in self.records)
        total_cache = sum(r.cache_read_tokens for r in self.records)
        latencies = [r.latency_ms for r in self.records]
        return {
            "llm_calls": len(self.records),
            "tool_calls": self.tool_call_count,
            "total_input_tokens": total_in,
            "total_output_tokens": total_out,
            "total_cache_read_tokens": total_cache,
            "avg_latency_ms": round(sum(latencies) / len(latencies), 1),
            "max_latency_ms": max(latencies),
            "per_call": [
                {"call": i + 1, "in": r.input_tokens, "out": r.output_tokens,
                 "latency_ms": r.latency_ms, "stop": r.stop_reason}
                for i, r in enumerate(self.records)
            ],
        }

# Usage
metrics = MetricsHook()
agent = Agent(model=BedrockModel(...), hooks=[metrics])
agent("Summarise the history of Rome in two paragraphs.")
print(metrics.summary())
```

---

## Safety / Guardrail Hook — Custom Content Filtering

Inspects user prompts and LLM outputs for blocked content without relying on AWS Bedrock Guardrails. Raises a custom exception on violation so the caller can handle it cleanly.

```python
import re
from strands.hooks import AgentHook

class ContentPolicyViolation(Exception):
    def __init__(self, stage: str, reason: str):
        super().__init__(f"[{stage}] Content policy violation: {reason}")
        self.stage = stage
        self.reason = reason

class SafetyGuardrailHook(AgentHook):
    """Keyword and regex-based content filter applied to both input and output."""

    BLOCKED_INPUT_PATTERNS = [
        re.compile(r"\b(how to (make|build|synthesize) (a bomb|explosives|weapons))\b", re.I),
        re.compile(r"\b(social security number|SSN)\b", re.I),
        re.compile(r"\b(credit card number)\b", re.I),
    ]

    BLOCKED_OUTPUT_PATTERNS = [
        re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),          # SSN pattern
        re.compile(r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b"),  # credit card
    ]

    def on_llm_call(self, request: dict) -> dict:
        # Check the most recent user message
        messages = request.get("messages", [])
        if messages:
            last = messages[-1]
            for block in last.get("content", []):
                text = block.get("text", "")
                for pattern in self.BLOCKED_INPUT_PATTERNS:
                    if pattern.search(text):
                        raise ContentPolicyViolation("input", f"matched pattern: {pattern.pattern}")
        return request

    def on_llm_response(self, response: dict) -> dict:
        # Check the assistant reply
        message = response.get("output", {}).get("message", {})
        for block in message.get("content", []):
            text = block.get("text", "")
            for pattern in self.BLOCKED_OUTPUT_PATTERNS:
                if pattern.search(text):
                    raise ContentPolicyViolation("output", f"matched pattern: {pattern.pattern}")
        return response

# Usage
try:
    agent = Agent(model=BedrockModel(...), hooks=[SafetyGuardrailHook()])
    result = agent("What is the capital of France?")
except ContentPolicyViolation as e:
    print(f"Blocked at {e.stage}: {e.reason}")
```

---

## Retry Hook — Intercept Errors and Retry LLM Calls

Intercepts `on_error` and retries the full invocation with exponential backoff. Because hooks cannot re-invoke the agent internally, the retry pattern wraps the agent call at the call site; the hook provides the retry logic and state.

```python
import asyncio
import time
import logging
from botocore.exceptions import ClientError
from strands.hooks import AgentHook

logger = logging.getLogger("strands.retry")

RETRYABLE_CODES = {"ThrottlingException", "ServiceUnavailableException", "InternalServerError"}

class RetryHook(AgentHook):
    """Records retryable errors; use with the retry_invoke helper below."""

    def __init__(self, max_attempts: int = 3, base_delay: float = 1.0):
        self.max_attempts = max_attempts
        self.base_delay = base_delay
        self.last_error: Exception | None = None

    def on_error(self, error: Exception) -> None:
        self.last_error = error

    def is_retryable(self, error: Exception) -> bool:
        if isinstance(error, ClientError):
            code = error.response["Error"]["Code"]
            return code in RETRYABLE_CODES
        return False


def retry_invoke(agent, prompt: str, hook: RetryHook) -> str:
    """Synchronous wrapper that retries on throttling / transient errors."""
    for attempt in range(hook.max_attempts):
        try:
            hook.last_error = None
            return str(agent(prompt))
        except Exception as e:
            if attempt == hook.max_attempts - 1 or not hook.is_retryable(e):
                raise
            delay = hook.base_delay * (2 ** attempt)
            logger.warning(f"Attempt {attempt + 1} failed ({type(e).__name__}), retrying in {delay}s")
            time.sleep(delay)
    raise RuntimeError("Retry loop exhausted without raising")


async def retry_invoke_async(agent, prompt: str, hook: RetryHook) -> str:
    """Async wrapper that retries on throttling / transient errors."""
    for attempt in range(hook.max_attempts):
        try:
            hook.last_error = None
            return str(await agent.invoke_async(prompt))
        except Exception as e:
            if attempt == hook.max_attempts - 1 or not hook.is_retryable(e):
                raise
            delay = hook.base_delay * (2 ** attempt)
            logger.warning(f"Async attempt {attempt + 1} failed, retrying in {delay}s")
            await asyncio.sleep(delay)
    raise RuntimeError("Retry loop exhausted without raising")


# Usage
retry_hook = RetryHook(max_attempts=4, base_delay=0.5)
agent = Agent(model=BedrockModel(...), hooks=[retry_hook])
result = retry_invoke(agent, "List the planets in the solar system.", retry_hook)
```

---

## Caching Hook — Cache LLM Responses for Identical Prompts

Caches the full LLM response for a given (system prompt, messages) fingerprint. Subsequent identical calls return the cached response without hitting the LLM. Useful in testing or when agents are called with repeated identical prompts.

```python
import hashlib
import json
from strands.hooks import AgentHook

class ResponseCacheHook(AgentHook):
    """In-memory response cache keyed on (model, system, messages) hash."""

    def __init__(self):
        self._cache: dict[str, dict] = {}
        self._pending_key: str | None = None
        self.hits: int = 0
        self.misses: int = 0

    def _fingerprint(self, request: dict) -> str:
        payload = {
            "model": request.get("model", ""),
            "system": request.get("system", []),
            "messages": request.get("messages", []),
        }
        raw = json.dumps(payload, sort_keys=True, default=str)
        return hashlib.sha256(raw.encode()).hexdigest()

    def on_llm_call(self, request: dict) -> dict:
        key = self._fingerprint(request)
        self._pending_key = key
        if key in self._cache:
            self.hits += 1
            # Signal to SDK: skip the real call and use cached response.
            # Attach the cached response directly onto the request so
            # on_llm_response can return it (SDK will use the returned value).
            request["__cached_response__"] = self._cache[key]
        else:
            self.misses += 1
        return request

    def on_llm_response(self, response: dict) -> dict:
        # If we injected a cached response, return it instead of the live one.
        # NOTE: This only works if the SDK uses the returned value from on_llm_response.
        if self._pending_key and self._pending_key not in self._cache:
            self._cache[self._pending_key] = response
        self._pending_key = None
        return response

    def cache_stats(self) -> dict:
        return {"size": len(self._cache), "hits": self.hits, "misses": self.misses}

    def invalidate(self) -> None:
        self._cache.clear()
        self.hits = 0
        self.misses = 0
```

> Note: Full short-circuit caching (skipping the LLM call entirely) requires the SDK to honour a non-None return from `on_llm_call` as a completed response. Check the installed Strands version for this capability. The pattern above is safe as a read-through cache that logs hit/miss rates regardless.

---

## Rate Limiting Hook

Enforces a maximum number of LLM calls per time window. Raises `RateLimitExceeded` when the budget is exhausted. Thread-safe via a lock.

```python
import threading
import time
from strands.hooks import AgentHook

class RateLimitExceeded(Exception):
    pass

class RateLimitHook(AgentHook):
    """Token-bucket rate limiter: max `limit` LLM calls per `window_seconds`."""

    def __init__(self, limit: int = 10, window_seconds: float = 60.0):
        self.limit = limit
        self.window_seconds = window_seconds
        self._calls: list[float] = []
        self._lock = threading.Lock()

    def on_llm_call(self, request: dict) -> dict:
        now = time.monotonic()
        with self._lock:
            # Evict timestamps outside the current window
            cutoff = now - self.window_seconds
            self._calls = [t for t in self._calls if t > cutoff]
            if len(self._calls) >= self.limit:
                oldest = self._calls[0]
                retry_in = round(self.window_seconds - (now - oldest), 1)
                raise RateLimitExceeded(
                    f"Rate limit of {self.limit} calls per {self.window_seconds}s exceeded. "
                    f"Retry in {retry_in}s."
                )
            self._calls.append(now)
        return request

    def remaining(self) -> int:
        now = time.monotonic()
        with self._lock:
            cutoff = now - self.window_seconds
            active = [t for t in self._calls if t > cutoff]
        return max(0, self.limit - len(active))

# Usage
rate_limiter = RateLimitHook(limit=5, window_seconds=60.0)
agent = Agent(model=BedrockModel(...), hooks=[rate_limiter])

try:
    result = agent("Summarise the French Revolution.")
except RateLimitExceeded as e:
    print(f"Throttled: {e}")
```

---

## Audit Trail Hook — Log All Inputs and Outputs to File

Writes a complete, append-only JSONL audit log of every LLM call (inputs and outputs) to a file. Suitable for compliance, debugging, and replay.

```python
import json
import time
import threading
from pathlib import Path
from strands.hooks import AgentHook

class AuditTrailHook(AgentHook):
    """Append-only JSONL audit log of every LLM call and tool invocation."""

    def __init__(self, log_path: str, agent_name: str = "agent"):
        self.log_path = Path(log_path)
        self.agent_name = agent_name
        self._lock = threading.Lock()
        self._pending_request: dict | None = None
        # Ensure file exists
        self.log_path.parent.mkdir(parents=True, exist_ok=True)
        self.log_path.touch(exist_ok=True)

    def _append(self, record: dict) -> None:
        line = json.dumps(record, default=str) + "\n"
        with self._lock:
            with self.log_path.open("a") as f:
                f.write(line)

    def on_llm_call(self, request: dict) -> dict:
        self._pending_request = request
        self._append({
            "ts": time.time(),
            "event": "llm_call",
            "agent": self.agent_name,
            "model": request.get("model"),
            "message_count": len(request.get("messages", [])),
            "system_blocks": len(request.get("system", [])),
            "last_user_message": self._extract_last_user_text(request),
        })
        return request

    def on_llm_response(self, response: dict) -> dict:
        usage = response.get("usage", {})
        message = response.get("output", {}).get("message", {})
        assistant_text = " ".join(
            b.get("text", "") for b in message.get("content", []) if b.get("type") == "text"
        )
        self._append({
            "ts": time.time(),
            "event": "llm_response",
            "agent": self.agent_name,
            "stop_reason": response.get("stopReason"),
            "input_tokens": usage.get("inputTokens", 0),
            "output_tokens": usage.get("outputTokens", 0),
            "assistant_text_preview": assistant_text[:200],
        })
        return response

    def on_tool_use(self, tool_name: str, tool_input: dict) -> dict:
        self._append({
            "ts": time.time(),
            "event": "tool_use",
            "agent": self.agent_name,
            "tool": tool_name,
            "input": tool_input,
        })
        return tool_input

    def on_tool_result(self, tool_name: str, tool_result: dict) -> dict:
        self._append({
            "ts": time.time(),
            "event": "tool_result",
            "agent": self.agent_name,
            "tool": tool_name,
            "status": tool_result.get("status"),
        })
        return tool_result

    def on_error(self, error: Exception) -> None:
        self._append({
            "ts": time.time(),
            "event": "error",
            "agent": self.agent_name,
            "error_type": type(error).__name__,
            "error_message": str(error),
        })

    @staticmethod
    def _extract_last_user_text(request: dict) -> str:
        messages = request.get("messages", [])
        for msg in reversed(messages):
            if msg.get("role") == "user":
                for block in msg.get("content", []):
                    if block.get("type") == "text":
                        return block["text"][:300]
        return ""

# Usage
audit = AuditTrailHook(log_path="/var/log/my-agent/audit.jsonl", agent_name="support-bot")
agent = Agent(model=BedrockModel(...), hooks=[audit])
agent("How do I reset my password?")
# /var/log/my-agent/audit.jsonl now contains a JSONL record for each event
```

---

## Multiple Hooks

Hooks compose cleanly. Register them in the order you want them to fire.

```python
from strands import Agent
from strands.models import BedrockModel

metrics   = MetricsHook()
audit     = AuditTrailHook(log_path="/tmp/audit.jsonl", agent_name="prod-agent")
safety    = SafetyGuardrailHook()
rate_limiter = RateLimitHook(limit=20, window_seconds=60.0)
logging_hook = StructuredLoggingHook(agent_name="prod-agent")

agent = Agent(
    model=BedrockModel(
        model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
        region_name="us-east-1",
    ),
    hooks=[rate_limiter, safety, logging_hook, metrics, audit],
    #       ^^^^^^^^^  fires first (block before any LLM cost)
    #                  ^^^^^^  content check second
    #                          ^^^^^^^^^^^^ structured logs third
    #                                       ^^^^^^^  accumulate stats
    #                                                ^^^^^  write audit last
)
```

Recommended registration order for production:
1. Rate limiter (cheapest check; blocks before any API cost)
2. Safety/guardrail hook (block harmful content early)
3. Logging hook (records what was allowed through)
4. Metrics hook (measures actual LLM call latency)
5. Audit trail hook (full record written last)

---

## Testing Hooks in Isolation

You can unit-test hook logic without running a real agent. Construct the hook, call its methods with synthetic data, and assert on its state.

```python
import pytest

class TestMetricsHook:
    def test_counts_calls_and_tokens(self):
        hook = MetricsHook()

        # Simulate two LLM turns
        hook.on_llm_call({"messages": [], "model": "test"})
        hook.on_llm_response({
            "usage": {"inputTokens": 100, "outputTokens": 50, "cacheReadInputTokens": 0},
            "stopReason": "end_turn",
        })
        hook.on_tool_use("calculator", {"expression": "1+1"})
        hook.on_llm_call({"messages": [], "model": "test"})
        hook.on_llm_response({
            "usage": {"inputTokens": 150, "outputTokens": 80, "cacheReadInputTokens": 0},
            "stopReason": "end_turn",
        })

        s = hook.summary()
        assert s["llm_calls"] == 2
        assert s["tool_calls"] == 1
        assert s["total_input_tokens"] == 250
        assert s["total_output_tokens"] == 130


class TestSafetyGuardrailHook:
    def _make_request(self, text: str) -> dict:
        return {
            "model": "test",
            "messages": [{"role": "user", "content": [{"type": "text", "text": text}]}],
            "system": [],
        }

    def test_passes_benign_input(self):
        hook = SafetyGuardrailHook()
        request = self._make_request("What is the capital of France?")
        result = hook.on_llm_call(request)
        assert result is not None  # not blocked

    def test_blocks_harmful_input(self):
        hook = SafetyGuardrailHook()
        request = self._make_request("How to make a bomb step by step")
        with pytest.raises(ContentPolicyViolation) as exc_info:
            hook.on_llm_call(request)
        assert exc_info.value.stage == "input"


class TestRateLimitHook:
    def test_allows_calls_within_limit(self):
        hook = RateLimitHook(limit=3, window_seconds=10.0)
        for _ in range(3):
            hook.on_llm_call({"model": "test", "messages": []})
        assert hook.remaining() == 0

    def test_raises_when_exceeded(self):
        hook = RateLimitHook(limit=2, window_seconds=10.0)
        hook.on_llm_call({"model": "test", "messages": []})
        hook.on_llm_call({"model": "test", "messages": []})
        with pytest.raises(RateLimitExceeded):
            hook.on_llm_call({"model": "test", "messages": []})
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Hook method name typo (e.g. `on_llm_called`) | Exact names: `on_start`, `on_llm_call`, `on_llm_response`, `on_tool_use`, `on_tool_result`, `on_error`, `on_end` |
| Hook not firing | Confirm the hook instance is in the `hooks=[...]` list passed to `Agent(...)` |
| Mutating `request` but not returning it | `on_llm_call` and `on_tool_use` must `return` the (possibly modified) dict; a missing return means `None` and the mutation is discarded |
| Hook raises an unhandled exception, aborting the agent | Wrap hook logic in `try/except` and log the error; re-raise only for intentional blocking (safety hooks) |
| Using sync hook methods in an async agent | Define `on_*_async` variants; sync hooks are called as fallback but cannot `await` |
| Sharing mutable state across concurrent agents without locking | Use `threading.Lock()` around any shared collections (see `RateLimitHook`) |
| Registering the same hook instance on multiple agents | Hook instance state is shared across agents; create one instance per agent unless sharing is intentional |
| Expecting `on_end` to not fire on error | `on_end` always fires; check `invocation_context["error"]` to distinguish success from failure |
| Testing hooks by running real LLM calls | Call hook methods directly with synthetic dicts; no real agent needed for unit tests |
| Rate limiter not thread-safe | Use `threading.Lock` (sync) or `asyncio.Lock` (async) around the call log |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Hook methods never called | Hook not in `hooks=[]` list | Print `agent.hooks` to verify registration |
| `AttributeError: 'NoneType' object has no attribute ...` in hook | Hook returns `None` from a mutating method and caller dereferences result | Always `return request` / `return response` from mutating hook methods |
| Hook fires but modified request not used | Returning a new dict instead of mutating in place — both work, but ensure the returned dict is complete | Log the request at the start and end of `on_llm_call` to confirm shape |
| `on_error` fires but agent still proceeds | `on_error` is observation-only; to abort, raise inside `on_llm_call` or `on_tool_use` | Move blocking logic into the appropriate pre-call hook |
| Async hook method `on_llm_call_async` not called | Agent invoked synchronously via `agent(prompt)` not `await agent.invoke_async(prompt)` | Use `invoke_async` for async hooks |
| Audit log file not created | Parent directory does not exist | Call `Path(log_path).parent.mkdir(parents=True, exist_ok=True)` before opening the file |
| Rate limiter counts are wrong under load | Multiple threads bypassing the lock | Ensure all reads and writes to `_calls` are inside the `with self._lock:` block |
| `ContentPolicyViolation` not caught by caller | Exception raised inside hook propagates to the call site; caller must handle it | Wrap `agent(prompt)` in `try/except ContentPolicyViolation` |
| Metrics show `0` latency | `on_llm_call` not storing start time before `on_llm_response` reads it | Store start time in `self._call_start` in `on_llm_call`, read it in `on_llm_response` |
