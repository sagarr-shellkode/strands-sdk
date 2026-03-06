---
name: strands-sdk-metrics
description: "Use when tracking token usage per invocation, measuring latency, publishing agent metrics to CloudWatch, enabling OpenTelemetry tracing, or understanding cost per agent call. Triggers on: metrics, token usage, latency, CloudWatch, OpenTelemetry, trace, observability, cost tracking, result.usage."
---

# Metrics and Observability — Strands SDK

## Overview

Strands exposes token usage, latency, and trace data per invocation. Use hooks to capture metrics, or enable built-in OpenTelemetry support for structured observability.

## Quick Reference

| Metric | Access | Location |
|---|---|---|
| Input tokens | `result.usage["input_tokens"]` | After invocation |
| Output tokens | `result.usage["output_tokens"]` | After invocation |
| Cache read tokens | `result.usage["cache_read_input_tokens"]` | After invocation |
| Latency | `time.time()` around invocation | Custom measurement |
| Traces | `trace_attributes` on Agent | OpenTelemetry |

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
print(f"Input tokens:  {usage.get('input_tokens', 0)}")
print(f"Output tokens: {usage.get('output_tokens', 0)}")
print(f"Cache hit:     {usage.get('cache_read_input_tokens', 0)}")
print(f"Latency:       {latency_ms:.0f}ms")
```

## Metrics Hook (Log Every Call)

```python
from strands.hooks import AgentHook
import logging

logger = logging.getLogger("agent.metrics")

class MetricsHook(AgentHook):
    def __init__(self):
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.call_count = 0

    def on_llm_response(self, response):
        usage = response.get("usage", {})
        self.total_input_tokens += usage.get("input_tokens", 0)
        self.total_output_tokens += usage.get("output_tokens", 0)
        self.call_count += 1
        logger.info(f"LLM call #{self.call_count}: {usage}")

    def summary(self):
        return {
            "calls": self.call_count,
            "total_input_tokens": self.total_input_tokens,
            "total_output_tokens": self.total_output_tokens
        }

metrics = MetricsHook()
agent = Agent(model=BedrockModel(...), hooks=[metrics])
agent("Question 1")
agent("Question 2")
print(metrics.summary())
```

## CloudWatch Metrics

```python
import boto3

cloudwatch = boto3.client("cloudwatch", region_name="us-east-1")

def publish_metrics(usage: dict, agent_name: str):
    cloudwatch.put_metric_data(
        Namespace="StrandsAgents",
        MetricData=[
            {"MetricName": "InputTokens", "Value": usage.get("input_tokens", 0),
             "Unit": "Count", "Dimensions": [{"Name": "AgentName", "Value": agent_name}]},
            {"MetricName": "OutputTokens", "Value": usage.get("output_tokens", 0),
             "Unit": "Count", "Dimensions": [{"Name": "AgentName", "Value": agent_name}]},
        ]
    )
```

## OpenTelemetry Tracing

```python
from strands import Agent
from strands.models import BedrockModel

agent = Agent(
    model=BedrockModel(...),
    trace_attributes={
        "service.name": "my-agent",
        "deployment.environment": "production",
        "agent.version": "1.0.0"
    }
)
# Configure via env vars:
# OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318
# OTEL_SERVICE_NAME=my-agent
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| `result.usage` is None | Check model provider returns usage; not all providers do |
| CloudWatch costs too high | Batch metrics or reduce publish frequency |
| OTEL traces not appearing | Verify `OTEL_EXPORTER_OTLP_ENDPOINT` env var is set |
| Token count wrong for multi-tool agents | Count is per LLM call; tool-using agents make multiple calls |
