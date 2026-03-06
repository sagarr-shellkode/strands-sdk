---
name: strands-sdk-multiagent
description: Use when orchestrating multiple Strands agents, deciding between A2A, swarms, or agents-as-tools patterns, understanding multi-agent architecture options, or decomposing complex tasks across specialized agents. Triggers on: multiple agents, orchestrator, sub-agent, agent coordination, multi-agent architecture, agent collaboration.
---

# Multi-Agent Orchestration — Strands SDK

## Overview

Strands supports three multi-agent patterns. Choose based on the coupling and coordination needs of your task.

## Pattern Decision Guide

```
Need agents to talk to each other directly?        → A2A (strands-sdk-a2a)
Need many agents running same task in parallel?    → Swarms (strands-sdk-agent-swarms)
Need orchestrator to delegate sub-tasks?           → Agents as Tools (strands-sdk-agents-as-tools)
```

## Pattern Comparison

| Pattern | Communication | Parallelism | Best For |
|---|---|---|---|
| A2A | Bidirectional, protocol-based | Sequential or parallel | Peer collaboration, negotiation |
| Swarms | Central coordinator to many workers | High parallelism | Same task, multiple inputs/perspectives |
| Agents as Tools | Orchestrator calls sub-agents as tools | On-demand | Task decomposition, specialization |

## Common Architecture: Orchestrator + Specialists

```
OrchestratorAgent
├── calls ResearchAgent (as tool) → gathers info
├── calls AnalysisAgent (as tool) → processes info
└── calls WriterAgent   (as tool) → formats output
```

All three patterns are composable. An A2A setup can internally use swarms; agents-as-tools can call A2A agents.

## Related Skills

- `strands-sdk-a2a` — Agent-to-Agent protocol implementation
- `strands-sdk-agent-swarms` — Swarm patterns with asyncio.gather
- `strands-sdk-agents-as-tools` — Wrapping agents as @tool callables
