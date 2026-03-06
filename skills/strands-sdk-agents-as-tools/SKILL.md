---
name: strands-sdk-agents-as-tools
description: Use when wrapping a Strands agent as a callable @tool for another agent, building hierarchical agent systems where an orchestrator delegates to specialist sub-agents, composing agents into a pipeline, or creating dynamic specialist agents on demand. Triggers on: agent as tool, sub-agent, specialist agent, nested agent, delegate to agent, hierarchical agents.
---

# Agents as Tools — Strands SDK

## Overview

Wrap any agent as a `@tool` so an orchestrator agent can call it on demand. The orchestrator decides when to call the sub-agent based on the docstring — just like any other tool.

## Basic Pattern

```python
from strands import Agent, tool
from strands.models import BedrockModel

# Specialist sub-agents
research_agent = Agent(
    model=BedrockModel(...),
    system_prompt="You are a research specialist. Find relevant facts and data."
)

writer_agent = Agent(
    model=BedrockModel(...),
    system_prompt="You are a technical writer. Write clear, structured documents."
)

# Wrap each as a tool
@tool
async def research(topic: str) -> str:
    """Research a topic and return key facts.

    Args:
        topic: The topic to research

    Returns:
        Key facts and data about the topic
    """
    result = await research_agent.invoke_async(f"Research: {topic}")
    return str(result)

@tool
async def write_document(content_brief: str) -> str:
    """Write a structured document from a content brief.

    Args:
        content_brief: The outline and key points to include

    Returns:
        Formatted document
    """
    result = await writer_agent.invoke_async(f"Write document: {content_brief}")
    return str(result)

# Orchestrator uses both sub-agents
orchestrator = Agent(
    model=BedrockModel(...),
    tools=[research, write_document],
    system_prompt="You orchestrate research and writing tasks."
)

result = await orchestrator.invoke_async(
    "Create a technical brief on prompt caching techniques."
)
```

## State Isolation

Each sub-agent maintains its own conversation history. The orchestrator sees only the tool return value, not sub-agent internals.

```python
# Clear sub-agent state between unrelated tasks
research_agent.conversation_manager.clear()
```

## Dynamic Sub-Agent Creation

```python
@tool
async def analyze_with_persona(persona: str, question: str) -> str:
    """Analyze a question from a specific expert persona.

    Args:
        persona: Expert role (e.g., security engineer, product manager)
        question: Question to analyze

    Returns:
        Analysis from that persona viewpoint
    """
    specialist = Agent(
        model=BedrockModel(...),
        system_prompt=f"You are a {persona}. Answer from your expert perspective."
    )
    return str(await specialist.invoke_async(question))
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Sub-agent tool has no docstring | Add docstring — orchestrator uses it to decide when to call |
| Sub-agent shares state across calls | Call `conversation_manager.clear()` between unrelated uses |
| Orchestrator never calls sub-agent | Make docstring explicit about exactly when to use it |
| Sync sub-agent in async tool | Use `invoke_async` in async tools |
