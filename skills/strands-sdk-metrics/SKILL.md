---
name: strands-sdk-metrics
description: "Use when tracking token usage per invocation, measuring latency, publishing agent metrics to CloudWatch, enabling OpenTelemetry tracing, or understanding cost per agent call. Triggers on: metrics, token usage, latency, CloudWatch, OpenTelemetry, trace, observability, cost tracking, result.usage, Prometheus, Grafana, X-Ray, Datadog, token budget, cost attribution, alert thresholds, anomaly detection, multi-agent tracing, P95 latency, per-user cost, per-session cost."
---

# Metrics and Observability — Strands SDK

## Overview

Strands exposes token usage, latency, and trace data per invocation. This skill covers the full observability stack: raw token/latency capture, cost calculation, per-session and per-user tracking, percentile latency (P50/P95/P99), Prometheus export, Grafana dashboards, AWS X-Ray tracing, Datadog integration, custom span tracing for multi-agent systems, alert thresholds, anomaly detection, token budget enforcement, and cost attribution per agent type.

---

## Quick Reference

| Metric | Access | Notes |
|---|---|---|
| Input tokens | `result.usage["input_tokens"]` | Per LLM call |
| Output tokens | `result.usage["output_tokens"]` | Per LLM call |
| Cache read tokens | `result.usage["cache_read_input_tokens"]` | Prompt cache hits (Anthropic) |
| Cache write tokens | `result.usage["cache_write_input_tokens"]` | Prompt cache writes (Anthropic) |
| Latency (wall clock) | `time.time()` around invocation | Includes tool calls |
| Cost estimate | See cost formulas below | Calculated from token counts |
| OTEL traces | `trace_attributes` on Agent | Requires OTEL exporter configured |
| CloudWatch metrics | `boto3` put_metric_data | Published after each invocation |
| Prometheus metrics | `/metrics` endpoint | Via `prometheus_client` |
| X-Ray traces | AWS SDK + OTEL X-Ray exporter | Auto-instruments boto3 calls |
| Datadog metrics | `datadog` SDK or DogStatsD | Via statsd or OTLP |

---

## Token Usage and Latency

```python
import time
from strands import Agent
from strands.models import BedrockModel

agent = Agent(model=BedrockModel(...))

start = time.time()
result = agent("Explain quantum entanglement.")
latency_ms = (time.time() - start) * 1000

usage = result.usage
print(f"Input tokens:        {usage.get('input_tokens', 0)}")
print(f"Output tokens:       {usage.get('output_tokens', 0)}")
print(f"Cache read tokens:   {usage.get('cache_read_input_tokens', 0)}")
print(f"Cache write tokens:  {usage.get('cache_write_input_tokens', 0)}")
print(f"Latency:             {latency_ms:.0f}ms")
```

`result.usage` is a dict populated after every invocation. For tool-using agents it reflects the **last** LLM call; use the `MetricsHook` below to accumulate across all calls in a single session.

---

## Cost Calculation Formulas

Prices are in USD per 1,000,000 tokens (MTok) as of early 2026. Always verify against the provider's current pricing page.

### Anthropic (via Bedrock cross-region inference)

| Model | Input $/MTok | Output $/MTok | Cache Write $/MTok | Cache Read $/MTok |
|---|---|---|---|---|
| Claude Haiku 3 | $0.25 | $1.25 | $0.30 | $0.03 |
| Claude Haiku 4 (Haiku-4-5) | $0.80 | $4.00 | $1.00 | $0.08 |
| Claude Sonnet 3.5 | $3.00 | $15.00 | $3.75 | $0.30 |
| Claude Sonnet 4 | $3.00 | $15.00 | $3.75 | $0.30 |
| Claude Opus 4 | $15.00 | $75.00 | $18.75 | $1.50 |

Bedrock on-demand adds ~10% overhead vs direct Anthropic API. Cross-region inference (global.*) routes to the cheapest available region but uses the same price tier.

### Cost formula

```python
def calculate_cost(usage: dict, model: str = "claude-haiku-4") -> float:
    """
    Returns estimated USD cost for a single LLM call.
    usage dict keys: input_tokens, output_tokens,
                     cache_read_input_tokens, cache_write_input_tokens
    """
    PRICES = {
        "claude-haiku-3": {
            "input": 0.25, "output": 1.25,
            "cache_write": 0.30, "cache_read": 0.03,
        },
        "claude-haiku-4": {
            "input": 0.80, "output": 4.00,
            "cache_write": 1.00, "cache_read": 0.08,
        },
        "claude-sonnet-3-5": {
            "input": 3.00, "output": 15.00,
            "cache_write": 3.75, "cache_read": 0.30,
        },
        "claude-sonnet-4": {
            "input": 3.00, "output": 15.00,
            "cache_write": 3.75, "cache_read": 0.30,
        },
        "claude-opus-4": {
            "input": 15.00, "output": 75.00,
            "cache_write": 18.75, "cache_read": 1.50,
        },
    }
    p = PRICES.get(model, PRICES["claude-haiku-4"])
    M = 1_000_000
    cost = (
        usage.get("input_tokens", 0) * p["input"] / M
        + usage.get("output_tokens", 0) * p["output"] / M
        + usage.get("cache_write_input_tokens", 0) * p["cache_write"] / M
        + usage.get("cache_read_input_tokens", 0) * p["cache_read"] / M
    )
    return cost
```

### Amortised cost with prompt caching

When a long system prompt is cached, every subsequent call pays `cache_read` instead of `input`. Break-even point:

```
cache_write_tokens * cache_write_$/MTok
    < N * cache_write_tokens * (input_$/MTok - cache_read_$/MTok)

Solve for N:
N > cache_write_$/MTok / (input_$/MTok - cache_read_$/MTok)

Haiku-4: N > 1.00 / (0.80 - 0.08) = 1.39  →  break-even at 2 reuses
Sonnet-4: N > 3.75 / (3.00 - 0.30) = 1.39  →  break-even at 2 reuses
```

---

## Per-Session and Per-User Cost Tracking

```python
import time
import threading
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from strands.hooks import AgentHook


@dataclass
class CallRecord:
    timestamp: float
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int
    cache_write_tokens: int
    latency_ms: float
    cost_usd: float
    model: str


class SessionMetrics:
    """Accumulates metrics for one logical session (conversation thread)."""

    def __init__(self, session_id: str, user_id: str, model: str = "claude-haiku-4"):
        self.session_id = session_id
        self.user_id = user_id
        self.model = model
        self.calls: List[CallRecord] = []
        self._lock = threading.Lock()

    def record(self, usage: dict, latency_ms: float) -> CallRecord:
        cost = calculate_cost(usage, self.model)
        record = CallRecord(
            timestamp=time.time(),
            input_tokens=usage.get("input_tokens", 0),
            output_tokens=usage.get("output_tokens", 0),
            cache_read_tokens=usage.get("cache_read_input_tokens", 0),
            cache_write_tokens=usage.get("cache_write_input_tokens", 0),
            latency_ms=latency_ms,
            cost_usd=cost,
            model=self.model,
        )
        with self._lock:
            self.calls.append(record)
        return record

    def total_cost(self) -> float:
        return sum(r.cost_usd for r in self.calls)

    def total_tokens(self) -> dict:
        return {
            "input": sum(r.input_tokens for r in self.calls),
            "output": sum(r.output_tokens for r in self.calls),
            "cache_read": sum(r.cache_read_tokens for r in self.calls),
            "cache_write": sum(r.cache_write_tokens for r in self.calls),
        }

    def latencies(self) -> List[float]:
        return [r.latency_ms for r in self.calls]


class UserCostTracker:
    """Aggregates costs across multiple sessions per user."""

    def __init__(self):
        self._sessions: Dict[str, SessionMetrics] = {}
        self._lock = threading.Lock()

    def get_or_create(
        self, session_id: str, user_id: str, model: str = "claude-haiku-4"
    ) -> SessionMetrics:
        with self._lock:
            if session_id not in self._sessions:
                self._sessions[session_id] = SessionMetrics(session_id, user_id, model)
            return self._sessions[session_id]

    def user_total_cost(self, user_id: str) -> float:
        return sum(
            s.total_cost()
            for s in self._sessions.values()
            if s.user_id == user_id
        )

    def user_sessions(self, user_id: str) -> List[SessionMetrics]:
        return [s for s in self._sessions.values() if s.user_id == user_id]

    def report(self) -> dict:
        by_user: Dict[str, float] = defaultdict(float)
        for s in self._sessions.values():
            by_user[s.user_id] += s.total_cost()
        return dict(by_user)


# Usage
tracker = UserCostTracker()

session = tracker.get_or_create("sess-abc", user_id="user-123", model="claude-haiku-4")
start = time.time()
result = agent("Hello")
session.record(result.usage, (time.time() - start) * 1000)

print(f"User cost so far: ${tracker.user_total_cost('user-123'):.6f}")
```

---

## Metrics Hook (Accumulate Across Tool Calls)

A tool-using agent makes multiple LLM calls per `agent(prompt)` invocation. The hook captures each one independently.

```python
import time
import logging
from strands.hooks import AgentHook

logger = logging.getLogger("agent.metrics")


class MetricsHook(AgentHook):
    """
    Accumulates token counts and latency across all LLM calls
    within a single agent session.
    """

    def __init__(self, model: str = "claude-haiku-4"):
        self.model = model
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.total_cache_read_tokens = 0
        self.total_cache_write_tokens = 0
        self.total_cost_usd = 0.0
        self.call_count = 0
        self._call_start: Optional[float] = None
        self.latencies_ms: List[float] = []

    def on_llm_start(self, request):
        self._call_start = time.time()

    def on_llm_response(self, response):
        latency_ms = (time.time() - self._call_start) * 1000 if self._call_start else 0.0
        self.latencies_ms.append(latency_ms)

        usage = response.get("usage", {})
        self.total_input_tokens += usage.get("input_tokens", 0)
        self.total_output_tokens += usage.get("output_tokens", 0)
        self.total_cache_read_tokens += usage.get("cache_read_input_tokens", 0)
        self.total_cache_write_tokens += usage.get("cache_write_input_tokens", 0)
        self.total_cost_usd += calculate_cost(usage, self.model)
        self.call_count += 1

        logger.info(
            "llm_call",
            extra={
                "call_num": self.call_count,
                "usage": usage,
                "latency_ms": latency_ms,
                "cost_usd": self.total_cost_usd,
            },
        )

    def summary(self) -> dict:
        return {
            "calls": self.call_count,
            "total_input_tokens": self.total_input_tokens,
            "total_output_tokens": self.total_output_tokens,
            "total_cache_read_tokens": self.total_cache_read_tokens,
            "total_cache_write_tokens": self.total_cache_write_tokens,
            "total_cost_usd": self.total_cost_usd,
            "latencies_ms": self.latencies_ms,
        }


metrics = MetricsHook(model="claude-haiku-4")
agent = Agent(model=BedrockModel(...), hooks=[metrics])
agent("Question 1")
agent("Question 2")
print(metrics.summary())
```

---

## P50/P95/P99 Latency Tracking

```python
import statistics
from typing import List


def percentiles(latencies_ms: List[float]) -> dict:
    if not latencies_ms:
        return {"p50": 0, "p95": 0, "p99": 0, "min": 0, "max": 0, "mean": 0}
    s = sorted(latencies_ms)
    n = len(s)

    def pct(p: float) -> float:
        idx = (p / 100) * (n - 1)
        lo, hi = int(idx), min(int(idx) + 1, n - 1)
        return s[lo] + (s[hi] - s[lo]) * (idx - lo)

    return {
        "p50": round(pct(50), 1),
        "p95": round(pct(95), 1),
        "p99": round(pct(99), 1),
        "min": round(s[0], 1),
        "max": round(s[-1], 1),
        "mean": round(statistics.mean(s), 1),
    }


# After collecting calls via MetricsHook or SessionMetrics:
stats = percentiles(metrics.latencies_ms)
print(f"P50={stats['p50']}ms  P95={stats['p95']}ms  P99={stats['p99']}ms")
```

Use a rolling window for live services to avoid unbounded memory growth:

```python
from collections import deque

class RollingLatency:
    def __init__(self, window: int = 1000):
        self._window = deque(maxlen=window)

    def record(self, latency_ms: float):
        self._window.append(latency_ms)

    def stats(self) -> dict:
        return percentiles(list(self._window))
```

---

## CloudWatch Metrics

```python
import boto3
from typing import Optional


cloudwatch = boto3.client("cloudwatch", region_name="us-east-1")


def publish_cloudwatch(
    usage: dict,
    latency_ms: float,
    cost_usd: float,
    agent_name: str,
    agent_type: Optional[str] = None,
    user_id: Optional[str] = None,
):
    dimensions = [{"Name": "AgentName", "Value": agent_name}]
    if agent_type:
        dimensions.append({"Name": "AgentType", "Value": agent_type})
    if user_id:
        dimensions.append({"Name": "UserId", "Value": user_id})

    cloudwatch.put_metric_data(
        Namespace="StrandsAgents",
        MetricData=[
            {
                "MetricName": "InputTokens",
                "Value": usage.get("input_tokens", 0),
                "Unit": "Count",
                "Dimensions": dimensions,
            },
            {
                "MetricName": "OutputTokens",
                "Value": usage.get("output_tokens", 0),
                "Unit": "Count",
                "Dimensions": dimensions,
            },
            {
                "MetricName": "CacheReadTokens",
                "Value": usage.get("cache_read_input_tokens", 0),
                "Unit": "Count",
                "Dimensions": dimensions,
            },
            {
                "MetricName": "LatencyMs",
                "Value": latency_ms,
                "Unit": "Milliseconds",
                "Dimensions": dimensions,
            },
            {
                "MetricName": "CostUSD",
                "Value": cost_usd,
                "Unit": "None",
                "Dimensions": dimensions,
            },
        ],
    )
```

### CloudWatch Alarms

```python
def create_latency_alarm(agent_name: str, threshold_ms: float = 5000):
    cloudwatch.put_metric_alarm(
        AlarmName=f"{agent_name}-HighLatency",
        MetricName="LatencyMs",
        Namespace="StrandsAgents",
        Statistic="p99",
        Period=300,          # 5-minute window
        EvaluationPeriods=2,
        Threshold=threshold_ms,
        ComparisonOperator="GreaterThanThreshold",
        Dimensions=[{"Name": "AgentName", "Value": agent_name}],
        AlarmActions=["arn:aws:sns:us-east-1:123456789012:agent-alerts"],
        TreatMissingData="notBreaching",
    )


def create_cost_alarm(agent_name: str, daily_budget_usd: float = 10.0):
    """Alert when daily spend exceeds budget."""
    cloudwatch.put_metric_alarm(
        AlarmName=f"{agent_name}-DailyBudgetExceeded",
        MetricName="CostUSD",
        Namespace="StrandsAgents",
        Statistic="Sum",
        Period=86400,        # 24-hour window
        EvaluationPeriods=1,
        Threshold=daily_budget_usd,
        ComparisonOperator="GreaterThanThreshold",
        Dimensions=[{"Name": "AgentName", "Value": agent_name}],
        AlarmActions=["arn:aws:sns:us-east-1:123456789012:agent-alerts"],
    )
```

---

## OpenTelemetry Tracing

### Setup

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME

resource = Resource.create({SERVICE_NAME: "strands-agent", "deployment.environment": "production"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4318/v1/traces"))
)
trace.set_tracer_provider(provider)

# Configure Strands agent to attach trace attributes
from strands import Agent
from strands.models import BedrockModel

agent = Agent(
    model=BedrockModel(...),
    trace_attributes={
        "service.name": "my-agent",
        "deployment.environment": "production",
        "agent.version": "1.0.0",
        "agent.type": "qa-bot",
    },
)
```

Required environment variables (alternative to SDK config):

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
OTEL_SERVICE_NAME=my-agent
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,agent.version=1.0.0
OTEL_TRACES_SAMPLER=traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1   # 10% sampling in high-volume production
```

---

## AWS X-Ray Tracing Integration

X-Ray is the native AWS distributed tracing service. Use the OTEL X-Ray exporter so Strands traces appear in the X-Ray console alongside Lambda, API Gateway, and Bedrock service maps.

```bash
pip install opentelemetry-sdk \
            opentelemetry-exporter-otlp-proto-grpc \
            aws-opentelemetry-distro \
            opentelemetry-propagator-aws-xray
```

```python
import boto3
from opentelemetry import trace, propagate
from opentelemetry.sdk.trace import TracerProvider, sampling
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.propagators.aws import AwsXRayPropagator
from opentelemetry.sdk.extension.aws.trace import AwsXRayIdGenerator
from opentelemetry.sdk.resources import Resource, SERVICE_NAME

# X-Ray requires its own ID format (trace IDs start with Unix timestamp)
resource = Resource.create({SERVICE_NAME: "strands-agent"})
provider = TracerProvider(
    resource=resource,
    id_generator=AwsXRayIdGenerator(),
)

# Send to AWS Distro for OpenTelemetry (ADOT) Collector sidecar
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
    )
)
trace.set_tracer_provider(provider)

# X-Ray propagation for cross-service correlation
propagate.set_global_textmap(AwsXRayPropagator())

tracer = trace.get_tracer("strands.agent")


def invoke_with_xray(agent, prompt: str, session_id: str) -> dict:
    with tracer.start_as_current_span(
        "agent.invoke",
        attributes={
            "agent.session_id": session_id,
            "agent.prompt_length": len(prompt),
        },
    ) as span:
        import time
        start = time.time()
        result = agent(prompt)
        latency_ms = (time.time() - start) * 1000

        usage = result.usage or {}
        span.set_attributes({
            "llm.input_tokens": usage.get("input_tokens", 0),
            "llm.output_tokens": usage.get("output_tokens", 0),
            "llm.latency_ms": latency_ms,
            "llm.cost_usd": calculate_cost(usage),
        })
        return result
```

### ADOT Collector config snippet (`otel-config.yaml`)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  awsxray:
    region: us-east-1
  awsemf:
    namespace: StrandsAgents
    region: us-east-1
    log_group_name: /strands/agents
    dimension_rollup_option: NoDimensionRollup

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [awsxray]
    metrics:
      receivers: [otlp]
      exporters: [awsemf]
```

---

## Prometheus Metrics Export

```bash
pip install prometheus_client
```

```python
from prometheus_client import (
    Counter, Histogram, Gauge, Summary,
    start_http_server, CollectorRegistry, REGISTRY,
)
import time

# Use a custom registry to avoid conflicts in multi-process environments
registry = CollectorRegistry()

INPUT_TOKENS = Counter(
    "strands_agent_input_tokens_total",
    "Total input tokens consumed",
    ["agent_name", "agent_type", "model"],
    registry=registry,
)
OUTPUT_TOKENS = Counter(
    "strands_agent_output_tokens_total",
    "Total output tokens consumed",
    ["agent_name", "agent_type", "model"],
    registry=registry,
)
COST_USD = Counter(
    "strands_agent_cost_usd_total",
    "Total estimated cost in USD",
    ["agent_name", "agent_type", "model"],
    registry=registry,
)
LATENCY = Histogram(
    "strands_agent_latency_ms",
    "End-to-end agent invocation latency in milliseconds",
    ["agent_name", "agent_type"],
    buckets=[100, 250, 500, 1000, 2000, 5000, 10000, 30000],
    registry=registry,
)
ACTIVE_SESSIONS = Gauge(
    "strands_agent_active_sessions",
    "Number of currently active agent sessions",
    ["agent_name"],
    registry=registry,
)
BUDGET_REMAINING = Gauge(
    "strands_agent_budget_remaining_usd",
    "Remaining token budget in USD",
    ["agent_name", "user_id"],
    registry=registry,
)


def record_prometheus_metrics(
    usage: dict,
    latency_ms: float,
    agent_name: str,
    agent_type: str,
    model: str,
):
    labels = {"agent_name": agent_name, "agent_type": agent_type, "model": model}
    INPUT_TOKENS.labels(**labels).inc(usage.get("input_tokens", 0))
    OUTPUT_TOKENS.labels(**labels).inc(usage.get("output_tokens", 0))
    COST_USD.labels(**labels).inc(calculate_cost(usage, model))
    LATENCY.labels(agent_name=agent_name, agent_type=agent_type).observe(latency_ms)


# Expose /metrics endpoint (standalone process)
start_http_server(9090, registry=registry)


# FastAPI integration
from fastapi import FastAPI, Response
from prometheus_client import generate_latest, CONTENT_TYPE_LATEST

app = FastAPI()

@app.get("/metrics")
def metrics():
    return Response(generate_latest(registry), media_type=CONTENT_TYPE_LATEST)
```

---

## Grafana Dashboard Setup

### Data source

Add Prometheus data source in Grafana pointing to `http://prometheus:9090`.

### Dashboard JSON panels (paste into "Import dashboard" → JSON model)

**Panel 1 — Tokens per minute**
```
sum(rate(strands_agent_input_tokens_total[1m])) by (agent_name)
  + sum(rate(strands_agent_output_tokens_total[1m])) by (agent_name)
```

**Panel 2 — Cost per hour**
```
sum(rate(strands_agent_cost_usd_total[1h])) by (agent_name) * 3600
```

**Panel 3 — P50/P95/P99 latency**
```
histogram_quantile(0.50, sum(rate(strands_agent_latency_ms_bucket[5m])) by (le, agent_name))
histogram_quantile(0.95, sum(rate(strands_agent_latency_ms_bucket[5m])) by (le, agent_name))
histogram_quantile(0.99, sum(rate(strands_agent_latency_ms_bucket[5m])) by (le, agent_name))
```

**Panel 4 — Active sessions**
```
strands_agent_active_sessions
```

**Panel 5 — Budget remaining by user**
```
strands_agent_budget_remaining_usd
```

### Grafana alert rule (P99 latency)

```yaml
# grafana/alerts/latency.yaml
apiVersion: 1
groups:
  - name: strands-agent-alerts
    interval: 1m
    rules:
      - uid: agent-p99-latency
        title: "Agent P99 latency > 8s"
        condition: C
        data:
          - refId: A
            queryType: ''
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: prometheus
            model:
              expr: histogram_quantile(0.99, sum(rate(strands_agent_latency_ms_bucket[5m])) by (le))
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              conditions:
                - evaluator:
                    params: [8000]
                    type: gt
        noDataState: NoData
        execErrState: Error
        for: 2m
        annotations:
          summary: "P99 agent latency has exceeded 8 seconds for 2 minutes"
```

---

## Datadog Integration

### Option A — DogStatsD (UDP, low overhead)

```bash
pip install datadog
```

```python
from datadog import initialize, statsd

initialize(statsd_host="localhost", statsd_port=8125)


def record_datadog(
    usage: dict,
    latency_ms: float,
    cost_usd: float,
    agent_name: str,
    agent_type: str,
    model: str,
    user_id: str = "anonymous",
):
    tags = [
        f"agent_name:{agent_name}",
        f"agent_type:{agent_type}",
        f"model:{model}",
        f"user_id:{user_id}",
    ]
    statsd.increment("strands.agent.input_tokens",
                     value=usage.get("input_tokens", 0), tags=tags)
    statsd.increment("strands.agent.output_tokens",
                     value=usage.get("output_tokens", 0), tags=tags)
    statsd.gauge("strands.agent.cost_usd", cost_usd, tags=tags)
    statsd.histogram("strands.agent.latency_ms", latency_ms, tags=tags)
```

### Option B — OTLP to Datadog Agent

Set environment variables so the OTEL SDK sends directly to Datadog:

```bash
DD_SITE=datadoghq.com
DD_API_KEY=<your-key>
OTEL_EXPORTER_OTLP_ENDPOINT=http://datadog-agent:4318
OTEL_SERVICE_NAME=strands-agent
```

The Datadog Agent (v7.44+) accepts OTLP traffic on port 4318 (HTTP) or 4317 (gRPC) and forwards traces to Datadog APM automatically.

### Datadog monitor example (cost anomaly)

```python
from datadog_api_client import ApiClient, Configuration
from datadog_api_client.v1.api.monitors_api import MonitorsApi
from datadog_api_client.v1.model.monitor import Monitor
from datadog_api_client.v1.model.monitor_type import MonitorType

config = Configuration()
with ApiClient(config) as api_client:
    api = MonitorsApi(api_client)
    api.create_monitor(
        Monitor(
            name="Strands Agent — Anomalous Cost Spike",
            type=MonitorType("metric alert"),
            query="anomalies(avg:strands.agent.cost_usd{*}.as_count(), 'agile', 3) >= 1",
            message="@slack-agent-alerts Cost anomaly detected on Strands agents.",
            tags=["service:strands-agent", "env:production"],
        )
    )
```

---

## Custom Span Tracing for Multi-Agent Systems

In orchestrator → sub-agent patterns, propagate the trace context so the entire call tree is visible in one trace.

```python
from opentelemetry import trace, context, baggage
from opentelemetry.propagate import inject, extract
import json

tracer = trace.get_tracer("strands.multi-agent")


class TracedOrchestrator:
    """
    Wraps an orchestrator agent and injects trace context into
    every sub-agent invocation so all spans share the same trace ID.
    """

    def __init__(self, orchestrator_agent, sub_agents: dict):
        self.orchestrator = orchestrator_agent
        self.sub_agents = sub_agents  # {"researcher": agent, "writer": agent, ...}

    def invoke(self, prompt: str, session_id: str):
        with tracer.start_as_current_span(
            "orchestrator.invoke",
            attributes={"session.id": session_id, "prompt.length": len(prompt)},
        ) as root_span:
            # Capture context for propagation to sub-agents
            carrier: dict = {}
            inject(carrier)

            results = {}
            for name, agent in self.sub_agents.items():
                results[name] = self._invoke_sub_agent(name, agent, prompt, carrier)

            root_span.set_attribute("sub_agents.count", len(results))
            return results

    def _invoke_sub_agent(self, name: str, agent, prompt: str, carrier: dict):
        # Restore parent context so this span is a child of orchestrator span
        ctx = extract(carrier)
        with tracer.start_as_current_span(
            f"sub_agent.{name}",
            context=ctx,
            attributes={"agent.name": name},
        ) as span:
            import time
            start = time.time()
            result = agent(prompt)
            latency_ms = (time.time() - start) * 1000

            usage = result.usage or {}
            span.set_attributes({
                "llm.input_tokens": usage.get("input_tokens", 0),
                "llm.output_tokens": usage.get("output_tokens", 0),
                "llm.latency_ms": latency_ms,
                "llm.cost_usd": calculate_cost(usage),
            })
            return result
```

---

## Alert Thresholds and Anomaly Detection

### Static thresholds

| Signal | Warning | Critical | Suggested action |
|---|---|---|---|
| P99 latency | > 5 s | > 10 s | Scale replicas; check Bedrock throttling |
| P95 latency | > 3 s | > 6 s | Review prompt length; enable caching |
| Cost/hour | > $1 | > $5 | Check runaway loops; enforce budget |
| Input tokens/call | > 50k | > 100k | Truncate context; use summarisation |
| Error rate | > 1% | > 5% | Check `AccessDeniedException`; retry logic |

### Z-score anomaly detection (rolling baseline)

```python
import math
from collections import deque


class AnomalyDetector:
    """
    Raises an alert when a metric deviates more than `z_threshold`
    standard deviations from its rolling mean.
    """

    def __init__(self, window: int = 100, z_threshold: float = 3.0):
        self._window = deque(maxlen=window)
        self._z_threshold = z_threshold

    def check(self, value: float, metric_name: str) -> bool:
        """Returns True if value is anomalous."""
        self._window.append(value)
        if len(self._window) < 10:
            return False  # not enough data

        mean = sum(self._window) / len(self._window)
        variance = sum((x - mean) ** 2 for x in self._window) / len(self._window)
        std = math.sqrt(variance) or 1e-9
        z = abs(value - mean) / std

        if z > self._z_threshold:
            import logging
            logging.getLogger("agent.anomaly").warning(
                f"ANOMALY: {metric_name}={value:.2f} is {z:.1f} std devs from mean={mean:.2f}"
            )
            return True
        return False


cost_anomaly = AnomalyDetector(window=200, z_threshold=3.0)
latency_anomaly = AnomalyDetector(window=200, z_threshold=3.5)

# In your metrics pipeline:
cost_anomaly.check(cost_usd, "cost_usd")
latency_anomaly.check(latency_ms, "latency_ms")
```

---

## Token Budget Enforcement

Stop an agent run — or raise an exception — when it would exceed a cost or token cap.

```python
from strands.hooks import AgentHook


class BudgetExceededError(RuntimeError):
    """Raised when the agent has consumed its token budget."""
    pass


class TokenBudgetHook(AgentHook):
    """
    Enforces a hard cost/token budget across all LLM calls
    within a single agent session. Raises BudgetExceededError
    before the next LLM call if the budget is exhausted.
    """

    def __init__(
        self,
        max_cost_usd: float = 0.10,
        max_input_tokens: int = 500_000,
        max_output_tokens: int = 100_000,
        model: str = "claude-haiku-4",
    ):
        self.max_cost_usd = max_cost_usd
        self.max_input_tokens = max_input_tokens
        self.max_output_tokens = max_output_tokens
        self.model = model

        self._total_cost_usd = 0.0
        self._total_input_tokens = 0
        self._total_output_tokens = 0

    @property
    def remaining_budget_usd(self) -> float:
        return self.max_cost_usd - self._total_cost_usd

    def on_llm_start(self, request):
        """Called before each LLM API call — enforce budget here."""
        if self._total_cost_usd >= self.max_cost_usd:
            raise BudgetExceededError(
                f"Token budget exceeded: spent ${self._total_cost_usd:.6f} "
                f"of ${self.max_cost_usd:.6f} limit."
            )
        if self._total_input_tokens >= self.max_input_tokens:
            raise BudgetExceededError(
                f"Input token budget exceeded: {self._total_input_tokens} "
                f"of {self.max_input_tokens} limit."
            )

    def on_llm_response(self, response):
        usage = response.get("usage", {})
        self._total_cost_usd += calculate_cost(usage, self.model)
        self._total_input_tokens += usage.get("input_tokens", 0)
        self._total_output_tokens += usage.get("output_tokens", 0)

    def status(self) -> dict:
        return {
            "spent_usd": self._total_cost_usd,
            "budget_usd": self.max_cost_usd,
            "remaining_usd": self.remaining_budget_usd,
            "pct_used": 100 * self._total_cost_usd / self.max_cost_usd,
            "input_tokens_used": self._total_input_tokens,
            "output_tokens_used": self._total_output_tokens,
        }


# Usage
budget = TokenBudgetHook(max_cost_usd=0.05, model="claude-haiku-4")
agent = Agent(model=BedrockModel(...), hooks=[budget])

try:
    result = agent("Run a complex multi-step analysis...")
except BudgetExceededError as e:
    print(f"Agent stopped: {e}")
    print(budget.status())
```

---

## Cost Attribution Per Agent Type

When running multiple agent types (orchestrator, researcher, writer, validator), break down cost by type for capacity planning.

```python
import threading
from collections import defaultdict
from typing import Dict


class CostAttributionTracker:
    """Thread-safe cost attribution across multiple agent types."""

    def __init__(self):
        self._costs: Dict[str, float] = defaultdict(float)
        self._tokens: Dict[str, Dict[str, int]] = defaultdict(lambda: defaultdict(int))
        self._calls: Dict[str, int] = defaultdict(int)
        self._lock = threading.Lock()

    def record(self, agent_type: str, usage: dict, model: str = "claude-haiku-4"):
        cost = calculate_cost(usage, model)
        with self._lock:
            self._costs[agent_type] += cost
            self._tokens[agent_type]["input"] += usage.get("input_tokens", 0)
            self._tokens[agent_type]["output"] += usage.get("output_tokens", 0)
            self._calls[agent_type] += 1

    def report(self) -> dict:
        with self._lock:
            total = sum(self._costs.values()) or 1e-9
            return {
                agent_type: {
                    "cost_usd": round(cost, 6),
                    "pct_of_total": round(100 * cost / total, 1),
                    "calls": self._calls[agent_type],
                    "avg_cost_per_call": round(cost / max(self._calls[agent_type], 1), 6),
                    "tokens": dict(self._tokens[agent_type]),
                }
                for agent_type, cost in sorted(
                    self._costs.items(), key=lambda x: -x[1]
                )
            }

    def total_cost(self) -> float:
        with self._lock:
            return sum(self._costs.values())


# Usage
attribution = CostAttributionTracker()

# In each agent's metrics hook:
attribution.record("orchestrator", result.usage, model="claude-sonnet-4")
attribution.record("researcher", result.usage, model="claude-haiku-4")
attribution.record("writer", result.usage, model="claude-sonnet-4")

import json
print(json.dumps(attribution.report(), indent=2))
# {
#   "writer":       {"cost_usd": 0.0024, "pct_of_total": 61.5, ...},
#   "orchestrator": {"cost_usd": 0.0009, "pct_of_total": 23.1, ...},
#   "researcher":   {"cost_usd": 0.0006, "pct_of_total": 15.4, ...}
# }
```

---

## Production Monitoring Checklist

Work through this before a production release.

### Instrumentation
- [ ] `MetricsHook` or equivalent attached to every Agent instance
- [ ] `TokenBudgetHook` configured with per-session cost cap
- [ ] `trace_attributes` set with `service.name`, `deployment.environment`, `agent.version`
- [ ] OTEL exporter endpoint configured (`OTEL_EXPORTER_OTLP_ENDPOINT`)
- [ ] All agents log `session_id` and `user_id` in every span

### CloudWatch
- [ ] Custom namespace `StrandsAgents` confirmed in CloudWatch Metrics console
- [ ] `InputTokens`, `OutputTokens`, `LatencyMs`, `CostUSD` metrics visible
- [ ] P99 latency alarm created and linked to SNS topic
- [ ] Daily budget alarm created per agent type

### Prometheus / Grafana
- [ ] `/metrics` endpoint returning valid Prometheus text format
- [ ] Prometheus scrape config includes agent service
- [ ] Grafana dashboard imported with token rate, cost, latency, active sessions panels
- [ ] Alert rules deployed for P99 latency and cost anomaly

### AWS X-Ray (if using Bedrock / Lambda)
- [ ] ADOT Collector sidecar running alongside agent service
- [ ] X-Ray service map shows agent → Bedrock call chain
- [ ] Sampling rate tuned (10% recommended for >100 RPS)

### Datadog (if applicable)
- [ ] DogStatsD or OTLP pipeline confirmed with `dd-agent status`
- [ ] `strands.agent.*` metrics visible in Metrics Explorer
- [ ] Anomaly monitor created for `cost_usd`

### Cost hygiene
- [ ] Model choice reviewed: use Haiku for high-volume, Sonnet/Opus only where quality requires it
- [ ] Prompt caching enabled for long system prompts (> 1024 tokens)
- [ ] Cache hit rate > 60% for repeated system prompts verified
- [ ] Attribution report reviewed weekly; top spender investigated

### Runbook
- [ ] On-call runbook documents how to raise/lower `TokenBudgetHook` limits at runtime
- [ ] Procedure for disabling a specific agent type without full redeploy documented
- [ ] Rollback steps for model version change documented

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `result.usage` is `None` | Provider does not return usage data | Verify model provider; Bedrock always returns usage for Anthropic models |
| Token count wrong for tool-using agents | Each LLM call in a tool loop has separate usage; `result.usage` reflects only the final call | Use `MetricsHook.on_llm_response` to accumulate across all calls |
| CloudWatch costs escalating | Too-frequent `put_metric_data` calls (billed per API call) | Batch up to 20 metrics per call; publish every 30 s instead of per-invocation |
| OTEL traces not appearing | Missing or wrong `OTEL_EXPORTER_OTLP_ENDPOINT` | Confirm the env var; test with `grpcurl` or `curl` to the collector endpoint |
| X-Ray traces missing Bedrock spans | boto3 not instrumented | Add `from opentelemetry.instrumentation.botocore import BotocoreInstrumentor; BotocoreInstrumentor().instrument()` before any boto3 calls |
| Datadog not receiving metrics | DogStatsD not running / wrong port | Run `dd-agent status`; confirm `statsd_host` and port 8125 are reachable |
| `BudgetExceededError` too early | Budget set too low for task complexity | Profile typical task cost with `MetricsHook.summary()` first; set budget to 3× P99 observed cost |
| High P99 latency (> 8 s) | Long context window or Bedrock throttling | Shorten system prompt; enable prompt caching; check Bedrock service quotas and request a limit increase |
| Cache hit rate is 0% | System prompt changes each call | Use a stable system prompt; move dynamic data to the user message |
| Cost attribution not adding up | Multiple agent types sharing one `MetricsHook` instance | Give each agent type its own `MetricsHook` instance tied to `CostAttributionTracker.record(agent_type, ...)` |
| Anomaly detector fires constantly | Window too small or `z_threshold` too low | Increase `window` to 200+; raise `z_threshold` to 3.5 for noisy signals |
| Prometheus histogram buckets mis-sized | Default Prometheus buckets are too coarse for ms-range latencies | Always specify explicit `buckets=[100, 250, 500, 1000, 2000, 5000, 10000, 30000]` |
