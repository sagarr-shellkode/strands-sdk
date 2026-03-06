---
name: strands-sdk-a2a
description: "Use when implementing Agent-to-Agent (A2A) communication, having one Strands agent call another agent over HTTP, setting up A2AServer or AgentClient, or building peer-to-peer distributed agent workflows. Triggers on: A2A, agent-to-agent, AgentClient, A2AServer, agent endpoint, inter-agent communication."
---

# Agent-to-Agent (A2A) — Strands SDK

## Overview

A2A allows agents to communicate over HTTP using a standard protocol. One agent acts as a server (exposes an endpoint), another as a client (calls that endpoint). Enables distributed, decoupled agent systems.

## Quick Reference

| Role | Class | Purpose |
|---|---|---|
| Server | `A2AServer` | Wraps an agent, exposes HTTP endpoint |
| Client | `AgentClient` | Calls a remote agent endpoint |
| Protocol | JSON over HTTP | Standard request/response format |

## Server Agent Setup

```python
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.a2a import A2AServer

specialist_agent = Agent(
    model=BedrockModel(...),
    system_prompt="You are a financial analysis specialist."
)

server = A2AServer(agent=specialist_agent, port=8080)
server.start()  # listens on http://localhost:8080
```

## Client Calling Remote Agent

```python
from strands import Agent, tool
from strands.models import BedrockModel
from strands.multiagent.a2a import AgentClient

@tool
async def call_financial_analyst(question: str) -> str:
    """Call the financial analysis specialist agent.

    Args:
        question: The financial question to analyze

    Returns:
        Analysis result from the specialist
    """
    client = AgentClient(url="http://localhost:8080")
    result = await client.invoke_async(question)
    return str(result)

orchestrator = Agent(
    model=BedrockModel(...),
    tools=[call_financial_analyst],
    system_prompt="You orchestrate tasks. Delegate financial questions to the analyst."
)

result = await orchestrator.invoke_async("Analyze the risk in this portfolio...")
```

## A2A with Authentication

```python
server = A2AServer(
    agent=specialist_agent,
    port=8080,
    api_key="your-secret-key"
)

client = AgentClient(
    url="http://remote-host:8080",
    api_key="your-secret-key"
)
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Server not started before client call | Ensure `server.start()` runs before client `invoke_async` |
| Port conflict | Change port or check for existing processes on that port |
| Firewall blocks inter-agent calls | Open the port in security group or firewall rules |
| Async client in sync context | Use `asyncio.run(client.invoke_async(...))` |
