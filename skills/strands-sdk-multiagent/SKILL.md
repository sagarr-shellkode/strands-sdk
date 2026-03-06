---
name: strands-sdk-multiagent
description: "Use when orchestrating multiple Strands agents, deciding between A2A, swarms, or agents-as-tools patterns, understanding multi-agent architecture options, or decomposing complex tasks across specialized agents. Triggers on: multiple agents, orchestrator, sub-agent, agent coordination, multi-agent architecture, agent collaboration."
---

# Multi-Agent Orchestration — Strands SDK

## Overview

Strands supports three first-class multi-agent patterns. The right choice depends on how tightly agents need to coordinate, whether they work in parallel or sequence, and whether communication is peer-to-peer or hierarchical.

All three patterns are composable: a supervisor built with agents-as-tools can fan out to a swarm; A2A endpoints can be wrapped as tools and called from an orchestrator.

---

## Pattern Decision Guide

```
Do agents need to talk directly to each other, peer-to-peer?
  → A2A  (strands-sdk-a2a)

Do many agents need to run the same (or similar) task simultaneously?
  → Swarms  (strands-sdk-agent-swarms)

Does one orchestrator need to delegate distinct sub-tasks to specialists?
  → Agents as Tools  (strands-sdk-agents-as-tools)
```

---

## Pattern Comparison

| Pattern | Communication | Topology | Parallelism | Best For |
|---|---|---|---|---|
| A2A | Bidirectional HTTP, protocol-based | Peer-to-peer or hub-and-spoke | Sequential or parallel | Distributed services, negotiation, cross-process agents |
| Swarms | Central coordinator fans out; workers respond | Star (coordinator + N workers) | High — all workers run concurrently | Same task on many inputs, multi-perspective analysis |
| Agents as Tools | Orchestrator calls sub-agents via `@tool` | Strict hierarchy | On-demand | Task decomposition, sequential pipelines, specialization |

### When to use A2A — concrete examples

Use A2A when agents live in separate processes, separate services, or separate machines and need to communicate at runtime:

- A compliance-checking agent that runs as a standalone HTTP service and is called by multiple different orchestrators across your organisation.
- A real-time negotiation loop where a buyer agent and a seller agent exchange offers back and forth until they converge.
- A microservice architecture where each agent team owns their own deployment lifecycle and you do not want a shared Python process.

```
Buyer Agent ──HTTP──► Seller Agent
      ◄──HTTP──────────────┘
  (offer/counter-offer loop)
```

### When to use Swarms — concrete examples

Use swarms when the same analytical task should be executed by multiple independent agents and their results combined:

- Run the same document through five agents with different reviewer personas (security, performance, cost, reliability, UX) simultaneously and aggregate the findings.
- Process 200 customer support tickets in parallel, each through the same classification agent, without waiting for one to finish before starting the next.
- Generate multiple candidate solutions to a problem and select the best via a synthesis agent.

```
                  ┌── ReviewerAgent (security)   ──┐
                  ├── ReviewerAgent (performance) ──┤
Coordinator ──►   ├── ReviewerAgent (cost)        ──┼──► SynthesisAgent
                  ├── ReviewerAgent (reliability) ──┤
                  └── ReviewerAgent (ux)           ──┘
```

### When to use Agents as Tools — concrete examples

Use agents-as-tools when an orchestrator must decide which specialist to invoke, in what order, and with what arguments:

- A research-writing pipeline: the orchestrator calls a ResearchAgent to gather facts, then a SummaryAgent to condense them, then a WriterAgent to produce a final document.
- A code-review system: the orchestrator calls a SyntaxAgent, a SecurityAgent, and a StyleAgent in sequence, then composes a consolidated review.
- A customer onboarding flow: the orchestrator calls a KYCAgent, then (on success) a ContractAgent, then a NotificationAgent.

```
OrchestratorAgent
├── research(topic) ──► ResearchAgent   → raw facts
├── summarise(facts) ─► SummaryAgent    → condensed brief
└── write(brief) ─────► WriterAgent     → final document
```

---

## Multi-Agent Design Principles

### Single Responsibility

Each agent should do exactly one thing well. Its system prompt defines its role; its tools give it capabilities for that role only. An agent that researches, writes, and sends emails is three agents poorly merged into one.

```python
# BAD — one agent, too many responsibilities
master = Agent(model=model, system_prompt="Research topics, write reports, and email stakeholders.")

# GOOD — each agent has one job
researcher = Agent(model=model, system_prompt="You find and summarise factual information on a topic.")
writer     = Agent(model=model, system_prompt="You turn structured notes into polished prose.")
notifier   = Agent(model=model, tools=[send_email], system_prompt="You send notifications via email.")
```

### Loose Coupling

Agents should communicate through well-defined interfaces (tool return values, HTTP responses), not through shared mutable state. Each agent receives inputs, produces outputs, and does not depend on the internal state of another agent.

- Prefer passing data explicitly as strings or structured dicts between agents.
- Avoid having multiple agents write to the same in-memory object concurrently.
- Design tool interfaces to be self-contained: the caller provides everything the sub-agent needs.

### Minimal Surface Area

The orchestrator should know as little as possible about how each sub-agent works internally. It calls a tool, gets a result, and moves on. Sub-agent conversation history, intermediate reasoning, and internal tool calls are invisible to the orchestrator — and should stay that way.

### Fail Gracefully

Every agent boundary is a potential failure point. Wrap inter-agent calls in error handling so a failure in one sub-agent does not silently corrupt the entire pipeline.

---

## State Sharing Between Agents

### Option 1 — Pass-Through (Recommended)

The simplest and safest approach: the orchestrator collects output from one agent and passes it as input to the next. No shared state, no concurrency hazards.

```python
@tool
async def research(topic: str) -> str:
    """Research a topic and return raw findings."""
    result = await research_agent.invoke_async(f"Research: {topic}")
    return str(result)

@tool
async def write_report(findings: str) -> str:
    """Write a report from structured findings."""
    result = await writer_agent.invoke_async(f"Write a report from these findings:\n{findings}")
    return str(result)

orchestrator = Agent(model=model, tools=[research, write_report],
                     system_prompt="Coordinate research and writing. Pass research output to the writer.")
```

### Option 2 — Shared External Store

For complex pipelines where multiple agents need to read and write the same data, use an external store (Redis, DynamoDB, S3, a local dict protected by a lock). Agents read from and write to the store by key; the orchestrator manages sequencing.

```python
import asyncio

pipeline_state: dict = {}
state_lock = asyncio.Lock()

@tool
async def store_result(key: str, value: str) -> str:
    """Store an intermediate result in the shared pipeline store."""
    async with state_lock:
        pipeline_state[key] = value
    return f"Stored '{key}'"

@tool
async def fetch_result(key: str) -> str:
    """Fetch an intermediate result from the shared pipeline store."""
    async with state_lock:
        return pipeline_state.get(key, "")
```

### Option 3 — Message Passing via A2A

For distributed agents in separate processes, pass state as structured JSON payloads in A2A request/response bodies. The receiving agent gets full context in the request; it writes nothing back to the caller's memory.

### Sub-Agent Conversation Isolation

Each sub-agent maintains its own conversation history. Between unrelated tasks, clear it to prevent context contamination:

```python
# After completing one pipeline run, reset sub-agent memory
research_agent.conversation_manager.clear()
writer_agent.conversation_manager.clear()
```

---

## Error Propagation in Multi-Agent Systems

Errors in sub-agents must be caught, categorised, and handled at every boundary. Without explicit handling, a single sub-agent failure silently returns an empty string or raises an exception that kills the orchestrator.

### Categorise Errors at Each Boundary

```python
from botocore.exceptions import ClientError
import logging

logger = logging.getLogger(__name__)

async def safe_agent_call(agent, prompt: str, agent_name: str, fallback: str = "") -> str:
    try:
        result = await agent.invoke_async(prompt)
        return str(result)
    except ClientError as e:
        code = e.response["Error"]["Code"]
        if code == "ThrottlingException":
            logger.warning(f"[{agent_name}] Rate limited — caller should retry with backoff")
            raise  # let caller retry
        elif code == "AccessDeniedException":
            logger.error(f"[{agent_name}] IAM permissions missing")
            raise  # fatal — do not swallow
        else:
            logger.error(f"[{agent_name}] AWS error {code}")
            return fallback
    except Exception as e:
        logger.error(f"[{agent_name}] Unexpected failure: {e}", exc_info=True)
        return fallback
```

### Retry at the Orchestrator Level

Sub-agents are stateless per-call — retrying is safe. Apply exponential backoff around rate-limited sub-agent calls:

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
                except ClientError as e:
                    if e.response["Error"]["Code"] != "ThrottlingException":
                        raise
                    if attempt == max_attempts - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    await asyncio.sleep(delay)
        return wrapper
    return decorator

@with_retry(max_attempts=3)
async def resilient_research(topic: str) -> str:
    return await safe_agent_call(research_agent, f"Research: {topic}", "ResearchAgent")
```

### Partial Failure in Swarms

When running `asyncio.gather`, use `return_exceptions=True` so one failed worker does not cancel all others. Filter results in the aggregator:

```python
async def robust_swarm(prompt: str, agents: list) -> list[str]:
    results = await asyncio.gather(
        *[agent.invoke_async(prompt) for agent in agents],
        return_exceptions=True
    )
    successes = []
    for i, r in enumerate(results):
        if isinstance(r, Exception):
            logger.warning(f"Worker {i} failed: {r}")
        else:
            successes.append(str(r))
    return successes
```

### Error Taxonomy for Multi-Agent Systems

| Error Type | Source | Handling Strategy |
|---|---|---|
| `ThrottlingException` | Bedrock rate limit | Retry with exponential backoff; reduce swarm concurrency |
| `AccessDeniedException` | IAM policy | Fail fast — misconfiguration, not transient |
| Sub-agent returns empty string | Model refused or bad prompt | Log + use fallback; do not pass empty output downstream |
| A2A connection refused | Remote agent not running | Fail the tool call with a clear error message |
| Pipeline state missing key | Sequencing bug | Assert required keys exist before passing to next agent |

---

## Orchestrator Patterns

### 1. Supervisor (Hierarchical Delegation)

A single orchestrator holds all decision-making authority. It calls specialists as tools and synthesises their outputs. Specialists are stateless from the orchestrator's perspective.

```
SupervisorAgent
├── calls ResearchAgent  → returns findings
├── calls AnalysisAgent  → returns structured analysis
└── calls WriterAgent    → returns final document
```

Best for: sequential pipelines, tasks with clear stage boundaries, when the orchestrator can determine the correct sequence up front.

```python
orchestrator = Agent(
    model=BedrockModel(model_id="anthropic.claude-sonnet-4-5-20251001-v1:0", region_name="us-east-1"),
    tools=[research, analyse, write_document],
    system_prompt="""You are a research orchestrator.
    For any request: first call research(), then analyse() on those findings, then write_document() with the analysis.
    Always complete all three steps in order."""
)
```

### 2. Pipeline (Linear Chain)

Each agent's output is the next agent's input. There is no central orchestrator — the pipeline is wired in code. Simpler than a supervisor but less flexible.

```
Input → AgentA → AgentB → AgentC → Output
```

Best for: well-defined ETL-style workflows, document transformation chains, when the sequence never changes.

```python
async def run_pipeline(input_text: str) -> str:
    # Stage 1: extract key entities
    entities = str(await extractor_agent.invoke_async(f"Extract entities: {input_text}"))
    extractor_agent.conversation_manager.clear()

    # Stage 2: enrich entities with external data
    enriched = str(await enricher_agent.invoke_async(f"Enrich these entities: {entities}"))
    enricher_agent.conversation_manager.clear()

    # Stage 3: generate final report
    report = str(await reporter_agent.invoke_async(f"Write a report: {enriched}"))
    reporter_agent.conversation_manager.clear()

    return report
```

### 3. Blackboard (Shared Workspace)

Multiple specialist agents read from and write to a shared workspace (the "blackboard"). A controller agent decides which specialist to activate next based on the current blackboard state. No specialist needs to know about the others.

```
Controller
    │ reads/writes
    ▼
┌──────────────────────┐
│   Blackboard (state) │
└──────────────────────┘
    ▲           ▲
SpecialistA   SpecialistB
(reads/writes) (reads/writes)
```

Best for: problems where the solution emerges incrementally, tasks requiring multiple passes by different experts over the same artifact (e.g., document review where each agent adds annotations).

```python
blackboard: dict = {"raw_doc": "", "entities": [], "sentiment": "", "summary": ""}

@tool
async def run_entity_extractor(doc: str) -> str:
    """Extract named entities from a document and store them."""
    result = str(await entity_agent.invoke_async(f"Extract entities from:\n{doc}"))
    blackboard["entities"] = result
    return result

@tool
async def run_sentiment_analyser(doc: str) -> str:
    """Analyse sentiment and store the result."""
    result = str(await sentiment_agent.invoke_async(f"Analyse sentiment of:\n{doc}"))
    blackboard["sentiment"] = result
    return result

controller = Agent(
    model=model,
    tools=[run_entity_extractor, run_sentiment_analyser],
    system_prompt="Process the document. Run entity extraction and sentiment analysis, then summarise both results."
)
```

---

## Monitoring Multi-Agent Systems

### Per-Agent Token and Latency Tracking

Attach a `MetricsHook` to each agent with a distinct label. Aggregate across the pipeline to get full-system cost visibility.

```python
import time
from strands.hooks import AgentHook

class NamedMetricsHook(AgentHook):
    def __init__(self, agent_name: str):
        self.agent_name = agent_name
        self.calls = 0
        self.input_tokens = 0
        self.output_tokens = 0
        self.total_latency_ms = 0.0
        self._call_start: float = 0.0

    def on_start(self, **kwargs):
        self._call_start = time.time()

    def on_llm_response(self, response, **kwargs):
        usage = response.get("usage", {})
        self.input_tokens  += usage.get("input_tokens", 0)
        self.output_tokens += usage.get("output_tokens", 0)
        self.calls += 1

    def on_end(self, **kwargs):
        self.total_latency_ms += (time.time() - self._call_start) * 1000

    def report(self) -> dict:
        return {
            "agent": self.agent_name,
            "calls": self.calls,
            "input_tokens": self.input_tokens,
            "output_tokens": self.output_tokens,
            "avg_latency_ms": round(self.total_latency_ms / max(self.calls, 1), 1),
        }

research_metrics  = NamedMetricsHook("ResearchAgent")
analysis_metrics  = NamedMetricsHook("AnalysisAgent")
writer_metrics    = NamedMetricsHook("WriterAgent")

research_agent = Agent(model=model, hooks=[research_metrics],  system_prompt="...")
analysis_agent = Agent(model=model, hooks=[analysis_metrics], system_prompt="...")
writer_agent   = Agent(model=model, hooks=[writer_metrics],   system_prompt="...")

def pipeline_report(hooks: list[NamedMetricsHook]) -> None:
    total_in = total_out = 0
    for h in hooks:
        r = h.report()
        print(f"  {r['agent']}: {r['calls']} calls, "
              f"{r['input_tokens']}in/{r['output_tokens']}out tokens, "
              f"{r['avg_latency_ms']}ms avg")
        total_in  += r["input_tokens"]
        total_out += r["output_tokens"]
    print(f"  TOTAL: {total_in} input / {total_out} output tokens")
```

### OpenTelemetry Distributed Tracing

Tag each agent with `trace_attributes` so traces across agents can be correlated in your observability backend (Jaeger, Grafana, AWS X-Ray via OTEL collector):

```python
PIPELINE_RUN_ID = "run-20260306-001"

research_agent = Agent(
    model=model,
    trace_attributes={
        "service.name": "research-agent",
        "pipeline.run_id": PIPELINE_RUN_ID,
        "agent.role": "research",
    }
)

writer_agent = Agent(
    model=model,
    trace_attributes={
        "service.name": "writer-agent",
        "pipeline.run_id": PIPELINE_RUN_ID,
        "agent.role": "writing",
    }
)
# OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318
# OTEL_SERVICE_NAME=multi-agent-pipeline
```

### Structured Logging Across Agent Boundaries

Log at every agent boundary with consistent fields so log queries can reconstruct the full pipeline execution:

```python
import logging, uuid, time

logger = logging.getLogger("pipeline")

async def traced_agent_call(agent, prompt: str, agent_name: str, run_id: str) -> str:
    call_id = str(uuid.uuid4())[:8]
    logger.info("agent_call_start", extra={
        "run_id": run_id, "call_id": call_id, "agent": agent_name,
        "prompt_len": len(prompt)
    })
    t0 = time.time()
    try:
        result = str(await agent.invoke_async(prompt))
        logger.info("agent_call_end", extra={
            "run_id": run_id, "call_id": call_id, "agent": agent_name,
            "latency_ms": round((time.time() - t0) * 1000),
            "result_len": len(result)
        })
        return result
    except Exception as e:
        logger.error("agent_call_error", extra={
            "run_id": run_id, "call_id": call_id, "agent": agent_name,
            "error": str(e)
        })
        raise
```

---

## Cost Considerations for Multi-Agent Setups

Every agent call costs tokens. A pipeline with three sequential agents costs three times as many tokens as a single agent, minimum — plus the overhead of passing context between stages.

### Cost Drivers

| Factor | Impact | Mitigation |
|---|---|---|
| Number of agents in pipeline | Linear increase in LLM calls | Only split when specialisation genuinely adds quality |
| Context passed between agents | Large inputs multiplied by agent count | Summarise before passing; strip irrelevant sections |
| Orchestrator system prompt size | Paid on every orchestrator LLM call | Keep orchestrator system prompt concise |
| Swarm width (N workers) | N × (input + output tokens) | Use smallest sufficient model for worker agents |
| Retry attempts | Multiplies cost on failure paths | Add retries only where failure is frequent; fix root causes |

### Model Tiering for Multi-Agent Pipelines

Use the most capable (and most expensive) model only where reasoning complexity demands it. Use cheaper models for mechanical or narrow tasks:

```python
from strands.models import BedrockModel

# Orchestrator: complex reasoning — use Sonnet
orchestrator_model = BedrockModel(
    model_id="anthropic.claude-sonnet-4-5-20251001-v1:0",
    region_name="us-east-1"
)

# Specialist workers: narrow, mechanical tasks — use Haiku
worker_model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

orchestrator    = Agent(model=orchestrator_model, tools=[research, analyse, write], system_prompt="...")
research_agent  = Agent(model=worker_model, system_prompt="Extract and summarise facts only.")
analysis_agent  = Agent(model=worker_model, system_prompt="Classify and score the provided data.")
```

### Cost Estimation Pattern

```python
# Approximate cost at time of writing (check current Bedrock pricing)
HAIKU_INPUT_PER_1K  = 0.00025   # USD per 1K input tokens
HAIKU_OUTPUT_PER_1K = 0.00125
SONNET_INPUT_PER_1K = 0.003
SONNET_OUTPUT_PER_1K = 0.015

def estimate_cost(input_tokens: int, output_tokens: int, model: str) -> float:
    if "haiku" in model:
        return (input_tokens / 1000) * HAIKU_INPUT_PER_1K + (output_tokens / 1000) * HAIKU_OUTPUT_PER_1K
    return (input_tokens / 1000) * SONNET_INPUT_PER_1K + (output_tokens / 1000) * SONNET_OUTPUT_PER_1K
```

---

## Testing Multi-Agent Systems

Testing multi-agent systems requires testing at three levels: unit (individual agents in isolation), integration (two agents interacting), and end-to-end (the full pipeline).

### Unit Testing Individual Agents

Mock the model to test agent logic — tool selection, prompt handling — without incurring API costs:

```python
import pytest
from unittest.mock import AsyncMock, MagicMock
from strands import Agent

def make_mock_agent(response_text: str) -> Agent:
    mock_model = MagicMock()
    mock_model.invoke_async = AsyncMock(return_value=MagicMock(__str__=lambda self: response_text))
    return Agent(model=mock_model, system_prompt="Test agent.")

@pytest.mark.asyncio
async def test_research_agent_returns_findings():
    agent = make_mock_agent("Key finding: X causes Y.")
    result = str(await agent.invoke_async("Research: what causes Y?"))
    assert "Key finding" in result
```

### Integration Testing Two-Agent Handoffs

Test that output from agent A is correctly structured to be consumed by agent B. Use real models only in integration tests — gate them behind a flag:

```python
import os
import pytest

REAL_LLM = os.getenv("INTEGRATION_TESTS") == "1"

@pytest.mark.skipif(not REAL_LLM, reason="Requires INTEGRATION_TESTS=1 and AWS credentials")
@pytest.mark.asyncio
async def test_research_to_writer_handoff():
    findings = str(await research_agent.invoke_async("Research: effects of sleep deprivation"))
    # Verify findings are non-empty and structurally suitable for the writer
    assert len(findings) > 50
    report = str(await writer_agent.invoke_async(f"Write a report:\n{findings}"))
    assert len(report) > 100
```

### End-to-End Pipeline Testing with Contract Assertions

Rather than asserting exact text (which varies with LLM outputs), assert structural contracts: required sections present, length thresholds met, no error markers in output:

```python
@pytest.mark.asyncio
async def test_full_pipeline_contract():
    result = await run_pipeline("Analyse the impact of remote work on productivity")
    # Contract assertions — not exact text matching
    assert len(result) >= 200, "Report too short — pipeline likely failed silently"
    assert "error" not in result.lower() or "no error" in result.lower()
    # Check for structural markers expected from the writer agent
    assert any(marker in result for marker in ["##", "Summary", "Conclusion", "Finding"])
```

### Testing Error Handling Paths

```python
@pytest.mark.asyncio
async def test_pipeline_handles_sub_agent_failure_gracefully():
    from botocore.exceptions import ClientError

    failing_agent = MagicMock()
    failing_agent.invoke_async = AsyncMock(side_effect=ClientError(
        {"Error": {"Code": "ThrottlingException", "Message": "Rate exceeded"}},
        "InvokeModel"
    ))

    result = await safe_agent_call(failing_agent, "test prompt", "TestAgent", fallback="FALLBACK")
    # ThrottlingException is re-raised for the caller to retry — fallback not used here
    # Test the swarm's partial failure path instead:
    results = await robust_swarm("test", [failing_agent])
    assert results == []  # failed worker excluded, no crash
```

### Testing Swarm Aggregation

```python
@pytest.mark.asyncio
async def test_swarm_aggregates_all_perspectives():
    mock_agents = [make_mock_agent(f"Perspective {i}: analysis text.") for i in range(3)]
    results = await asyncio.gather(*[a.invoke_async("prompt") for a in mock_agents])
    assert len(results) == 3
    assert all(str(r) for r in results)
```

---

## Common Multi-Agent Architectures (Text Diagrams)

### Research Pipeline

```
User Request
     │
     ▼
OrchestratorAgent (Sonnet)
     ├──► QueryFormulatorAgent (Haiku)
     │         └── returns: list of search queries
     ├──► [for each query] SearchAgent (Haiku)   ← swarm
     │         └── returns: raw search result
     ├──► DeduplicationAgent (Haiku)
     │         └── returns: deduplicated facts
     └──► ReportWriterAgent (Sonnet)
               └── returns: final report
```

### Code Review System

```
Pull Request Diff
     │
     ▼
ReviewOrchestratorAgent (Sonnet)
     ├──► SyntaxCheckerAgent (Haiku)   → syntax issues
     ├──► SecurityAuditAgent (Sonnet)  → security vulnerabilities
     ├──► StyleAgent (Haiku)           → style violations
     └──► ReviewAggregatorAgent (Haiku)
               └── merges all feedback → PR comment
```

### Document Processing System

```
Raw Document
     │
     ▼
ClassifierAgent (Haiku)
     │ → document type (invoice / contract / report)
     │
     ├──[invoice]──► InvoiceExtractorAgent (Haiku)
     │                    └── → structured line items
     ├──[contract]─► ContractAnalyserAgent (Sonnet)
     │                    └── → risk clauses, obligations
     └──[report]──► ReportSummaryAgent (Haiku)
                        └── → executive summary

(all paths) → StoreAgent (Haiku) → persists to database
```

### Multi-Perspective Decision Support

```
Question / Decision
        │
        ▼
DispatcherAgent
   │        │        │        │
   ▼        ▼        ▼        ▼
Optimist  Critic  Pragmatist  Devil's
Agent     Agent   Agent      Advocate Agent
   │        │        │        │
   └────────┴────────┴────────┘
                │
                ▼
         SynthesisAgent
                │
                ▼
      Balanced Recommendation
```

---

## Anti-Patterns

### God Agent

An agent with too many responsibilities and too many tools. It becomes hard to test, produces inconsistent outputs, and its system prompt grows unmanageably large.

```python
# BAD — god agent
agent = Agent(
    model=model,
    tools=[search_web, read_file, write_file, send_email, call_api,
           query_database, generate_image, translate_text, run_code],
    system_prompt="You can do everything. Handle any request."
)

# GOOD — split by capability domain
data_agent    = Agent(model=model, tools=[query_database, read_file], system_prompt="You retrieve data.")
action_agent  = Agent(model=model, tools=[send_email, call_api],      system_prompt="You trigger actions.")
content_agent = Agent(model=model, tools=[generate_image, translate_text], system_prompt="You process content.")
```

### Tight Coupling Between Agents

When one agent's internal prompt format is hardcoded in another agent's logic, any prompt change breaks the integration. Communicate through neutral interfaces (plain strings, JSON) not agent-implementation-specific formats.

```python
# BAD — writer agent expects exact research agent output format
@tool
async def write_report(research_output: str) -> str:
    # Brittle: assumes research_output starts with "FINDINGS:\n"
    findings = research_output.split("FINDINGS:\n")[1]
    ...

# GOOD — orchestrator normalises before passing
@tool
async def write_report(facts: str) -> str:
    """Write a report from a list of facts. facts is plain text."""
    result = await writer_agent.invoke_async(f"Write a structured report from:\n{facts}")
    return str(result)
```

### Shared Mutable State Between Concurrent Agents

Multiple swarm workers writing to the same Python dict or list without locking causes data races. In async Python, context switches can happen between any two `await` points.

```python
# BAD — race condition
results_list = []
async def worker(prompt):
    result = await agent.invoke_async(prompt)
    results_list.append(str(result))   # unsafe if multiple workers run concurrently

# GOOD — use asyncio.gather return values, not shared mutation
async def safe_swarm(prompts: list[str]) -> list[str]:
    agents = [Agent(model=model) for _ in prompts]
    results = await asyncio.gather(*[a.invoke_async(p) for a, p in zip(agents, prompts)])
    return [str(r) for r in results]
```

### Passing Entire Conversation History Between Agents

Each agent accumulates its own conversation history. Never dump one agent's full history into another's prompt — it blows up token counts and contaminates context.

```python
# BAD — passing conversation history across agents
history = str(research_agent.conversation_manager.messages)
writer_result = await writer_agent.invoke_async(f"Here is everything the research agent saw:\n{history}")

# GOOD — pass only the synthesised output
findings = str(await research_agent.invoke_async("Research: quantum computing advances in 2025"))
writer_result = await writer_agent.invoke_async(f"Write a report based on:\n{findings}")
```

### Ignoring Sub-Agent Failures

Swallowing exceptions from sub-agents and passing empty strings downstream produces garbage output with no error signal.

```python
# BAD
try:
    result = str(await agent.invoke_async(prompt))
except:
    result = ""   # silently passes "" downstream

# GOOD — surface the failure explicitly
try:
    result = str(await agent.invoke_async(prompt))
except Exception as e:
    logger.error(f"Sub-agent failed: {e}")
    raise  # or return a sentinel value the orchestrator can detect
```

### Infinite Delegation Loops

An orchestrator calls AgentA, which calls AgentB, which calls back to the orchestrator. With no termination condition, this recurses until token limits or timeouts kill the process.

Design rule: delegation is always a directed acyclic graph. An agent must never be reachable as a downstream tool of an agent it can call.

---

## Real-World Examples

### Research Pipeline (Full Implementation)

```python
import asyncio
from strands import Agent, tool
from strands.models import BedrockModel

model_fast = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",   region_name="us-east-1")
model_smart = BedrockModel(model_id="anthropic.claude-sonnet-4-5-20251001-v1:0", region_name="us-east-1")

# Specialists
searcher_agent = Agent(model=model_fast,  system_prompt="You extract and list factual statements on a topic. Be concise.")
analyst_agent  = Agent(model=model_smart, system_prompt="You analyse facts and identify key insights and gaps.")
writer_agent   = Agent(model=model_smart, system_prompt="You write clear, structured research briefs for a technical audience.")

@tool
async def search_topic(topic: str) -> str:
    """Search for factual information on a topic. Returns a list of facts."""
    result = str(await searcher_agent.invoke_async(f"List 10 key facts about: {topic}"))
    searcher_agent.conversation_manager.clear()
    return result

@tool
async def analyse_facts(facts: str) -> str:
    """Analyse a list of facts and extract the most important insights."""
    result = str(await analyst_agent.invoke_async(f"Analyse these facts and summarise key insights:\n{facts}"))
    analyst_agent.conversation_manager.clear()
    return result

@tool
async def write_brief(insights: str) -> str:
    """Write a structured research brief from a set of insights."""
    result = str(await writer_agent.invoke_async(f"Write a 3-section research brief from:\n{insights}"))
    writer_agent.conversation_manager.clear()
    return result

orchestrator = Agent(
    model=model_smart,
    tools=[search_topic, analyse_facts, write_brief],
    system_prompt="""You coordinate a research pipeline.
    For each request: 1) search_topic to gather facts, 2) analyse_facts on those facts, 3) write_brief on the insights.
    Always complete all three steps and return the final brief."""
)

async def main():
    result = await orchestrator.invoke_async("Research the current state of quantum error correction")
    print(str(result))

asyncio.run(main())
```

### Code Review System

```python
syntax_agent   = Agent(model=model_fast,  system_prompt="You identify syntax errors and undefined variables in code. List issues only.")
security_agent = Agent(model=model_smart, system_prompt="You identify security vulnerabilities: injection, auth issues, secrets in code.")
style_agent    = Agent(model=model_fast,  system_prompt="You identify style issues against PEP8 and clean code principles.")

@tool
async def check_syntax(code: str) -> str:
    """Check code for syntax errors and undefined references."""
    r = str(await syntax_agent.invoke_async(f"Check syntax:\n```\n{code}\n```"))
    syntax_agent.conversation_manager.clear()
    return r

@tool
async def check_security(code: str) -> str:
    """Audit code for security vulnerabilities."""
    r = str(await security_agent.invoke_async(f"Security audit:\n```\n{code}\n```"))
    security_agent.conversation_manager.clear()
    return r

@tool
async def check_style(code: str) -> str:
    """Check code for style and readability issues."""
    r = str(await style_agent.invoke_async(f"Style review:\n```\n{code}\n```"))
    style_agent.conversation_manager.clear()
    return r

review_orchestrator = Agent(
    model=model_smart,
    tools=[check_syntax, check_security, check_style],
    system_prompt="""You run a thorough code review.
    Run all three checks (syntax, security, style) and consolidate findings into a prioritised review comment.
    Label each issue as CRITICAL, HIGH, MEDIUM, or LOW."""
)
```

### Document Processing with Classifier Router

```python
invoice_agent  = Agent(model=model_fast,  system_prompt="Extract invoice fields: vendor, date, line items, total. Return JSON.")
contract_agent = Agent(model=model_smart, system_prompt="Identify obligations, parties, key dates, and risk clauses in contracts.")
summary_agent  = Agent(model=model_fast,  system_prompt="Summarise reports in 3 bullet points.")

@tool
async def classify_document(text: str) -> str:
    """Classify a document as: invoice, contract, or report."""
    classifier = Agent(model=model_fast, system_prompt="Respond with exactly one word: invoice, contract, or report.")
    result = str(await classifier.invoke_async(f"Classify this document:\n{text[:500]}"))
    return result.strip().lower()

@tool
async def process_invoice(text: str) -> str:
    """Extract structured data from an invoice."""
    r = str(await invoice_agent.invoke_async(f"Extract invoice data:\n{text}"))
    invoice_agent.conversation_manager.clear()
    return r

@tool
async def process_contract(text: str) -> str:
    """Analyse a contract for key terms and risks."""
    r = str(await contract_agent.invoke_async(f"Analyse contract:\n{text}"))
    contract_agent.conversation_manager.clear()
    return r

@tool
async def process_report(text: str) -> str:
    """Summarise a report document."""
    r = str(await summary_agent.invoke_async(f"Summarise:\n{text}"))
    summary_agent.conversation_manager.clear()
    return r

doc_router = Agent(
    model=model_fast,
    tools=[classify_document, process_invoice, process_contract, process_report],
    system_prompt="""You route documents to the correct processor.
    First classify_document, then call the matching processor (process_invoice / process_contract / process_report).
    Return the structured output from the processor."""
)
```

---

## Related Skills

- `strands-sdk-a2a` — Agent-to-Agent protocol: `A2AServer`, `AgentClient`, HTTP-based inter-agent calls
- `strands-sdk-agent-swarms` — Swarm patterns: `asyncio.gather`, fan-out/fan-in, concurrency limiting
- `strands-sdk-agents-as-tools` — Wrapping agents as `@tool` callables for orchestrators
- `strands-sdk-metrics` — Token tracking, latency, CloudWatch, OpenTelemetry for observability
- `strands-sdk-hooks` — Lifecycle hooks for logging, metrics, and request mutation across agents
- `strands-sdk-best-practices` — Error handling, retries, model selection, agent structure
