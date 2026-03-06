---
name: strands-sdk-tools
description: Use when defining custom tools with @tool decorator, using Strands built-in tools, integrating MCP tools with an agent, passing tools to Agent constructor, or debugging tool call failures. Triggers on: @tool, use_aws, file_read, http_request, tool not found, MCP, tool schema, built-in tools.
---

# Tools — Strands SDK

## Overview

Tools extend what an agent can do. Define Python functions with `@tool`, pass them to `Agent(tools=[...])`. Strands ships built-in tools and supports MCP tool servers.

## Quick Reference

| Category | Import | Example |
|---|---|---|
| Custom tool | `from strands import tool` | `@tool` decorator |
| Built-in: file | `from strands.tools import file_read, file_write` | read/write files |
| Built-in: HTTP | `from strands.tools import http_request` | make HTTP calls |
| Built-in: AWS | `from strands.tools import use_aws` | AWS SDK calls |
| Built-in: shell | `from strands.tools import shell` | run shell commands |
| MCP tools | `from strands.tools.mcp import MCPClient` | connect MCP server |

## Custom Tool Pattern

```python
from strands import Agent, tool
from strands.models import BedrockModel

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city.

    Args:
        city: Name of the city to get weather for

    Returns:
        Weather description string
    """
    return f"Sunny, 22C in {city}"

agent = Agent(model=BedrockModel(...), tools=[get_weather])
result = agent("What's the weather in Paris?")
```

## Tool Schema Rules

- Function **docstring is required** — used as tool description for the LLM
- Args section in docstring describes parameters to the LLM
- Type hints on parameters build the JSON schema
- Return type annotation improves reliability
- Keep function names descriptive (snake_case)

## Built-in Tools

```python
from strands.tools import file_read, file_write, http_request, shell, use_aws

agent = Agent(
    model=model,
    tools=[file_read, file_write, http_request]
)
```

## MCP Tool Integration

```python
from strands.tools.mcp import MCPClient

async with MCPClient({"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]}) as mcp:
    tools = await mcp.list_tools_async()
    agent = Agent(model=model, tools=tools)
    result = await agent.invoke_async("List files in /tmp")
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Missing docstring on `@tool` function | Add docstring — it is the tool description sent to the LLM |
| Tool not called by agent | Rephrase prompt to explicitly mention the tool's purpose |
| `use_aws` fails | Ensure AWS credentials are configured in the environment |
| Async tool in sync agent | Use `invoke_async` or make the tool synchronous |
