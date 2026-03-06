---
name: strands-sdk-agent-swarms
description: Use when running multiple Strands agents in parallel on the same or similar tasks, implementing fan-out/fan-in patterns, processing batches concurrently, or building swarm intelligence workflows with result aggregation. Triggers on: swarm, parallel agents, fan-out, asyncio.gather, batch processing, concurrent agents, fan-in synthesis.
---

# Agent Swarms — Strands SDK

## Overview

Swarms run many agents concurrently on the same or similar tasks, then aggregate results. Use Python's `asyncio.gather` to fan out to multiple agents and collect all responses.

## Quick Reference

| Pattern | Tool | Use Case |
|---|---|---|
| Fan-out | `asyncio.gather` | Same prompt, multiple agents |
| Fan-out/fan-in | `gather` + aggregator agent | Parallel work, then synthesis |
| Batch processing | `gather` over list | Different inputs, same agent type |
| Rate-limited swarm | `asyncio.Semaphore` | Cap concurrency, avoid throttling |

## Basic Swarm (Fan-Out)

```python
import asyncio
from strands import Agent
from strands.models import BedrockModel

async def swarm(prompt: str) -> list[str]:
    agents = [
        Agent(model=BedrockModel(...), system_prompt="You are optimistic. Find positives."),
        Agent(model=BedrockModel(...), system_prompt="You are critical. Find risks."),
        Agent(model=BedrockModel(...), system_prompt="You are pragmatic. Find trade-offs."),
    ]

    results = await asyncio.gather(
        *[agent.invoke_async(prompt) for agent in agents]
    )
    return [str(r) for r in results]

results = asyncio.run(swarm("Should we adopt microservices architecture?"))
```

## Fan-Out + Fan-In (Synthesis)

```python
async def swarm_with_synthesis(prompt: str) -> str:
    perspectives = await swarm(prompt)

    synthesizer = Agent(
        model=BedrockModel(...),
        system_prompt="Synthesize multiple perspectives into a balanced recommendation."
    )

    combined = "\n\n".join([f"View {i+1}: {p}" for i, p in enumerate(perspectives)])
    final = await synthesizer.invoke_async(
        f"Synthesize these perspectives on '{prompt}':\n\n{combined}"
    )
    return str(final)
```

## Concurrency Limit (Avoid Rate Limits)

```python
async def limited_swarm(items: list[str], max_concurrent: int = 5) -> list[str]:
    semaphore = asyncio.Semaphore(max_concurrent)

    async def run_one(item: str) -> str:
        async with semaphore:
            agent = Agent(model=BedrockModel(...))
            return str(await agent.invoke_async(item))

    return await asyncio.gather(*[run_one(item) for item in items])
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Sharing one Agent across concurrent calls | Create a new Agent per swarm worker |
| ThrottlingException from Bedrock | Use Semaphore to cap concurrency to 3-5 |
| Results out of order | `asyncio.gather` preserves input order |
| Memory spike with large swarms | Process in batches of N using `limited_swarm` |
