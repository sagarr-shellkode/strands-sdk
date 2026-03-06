# strands-sdk

A Claude Code plugin providing comprehensive knowledge of the Strands-agents Python SDK. This plugin auto-loads contextual reference material when you work with Strands-agents code, and exposes a `/strands-docs` command for on-demand lookups.

## Features

- **16 auto-triggering skills** that activate based on context (tools, hooks, multi-agent patterns, caching, guardrails, and more)
- **`/strands-docs` command** for on-demand documentation queries
- Covers the full Strands-agents lifecycle: from basic agent invocation to production observability
- Includes exact error messages and fixes in the troubleshooting skill

## Skills

| Skill | Covers |
|---|---|
| strands-sdk-core | Agent, BedrockModel, basic invocation |
| strands-sdk-tools | @tool decorator, built-ins, MCP |
| strands-sdk-prompt-caching | cache_point, caching strategies |
| strands-sdk-conversation | Multi-turn, history, context |
| strands-sdk-streaming | Streaming, callbacks, async |
| strands-sdk-hooks | Lifecycle hooks, before/after |
| strands-sdk-multiagent | Orchestration overview |
| strands-sdk-a2a | Agent-to-Agent protocol |
| strands-sdk-agent-swarms | Parallel swarms, fan-out/fan-in |
| strands-sdk-agents-as-tools | Agents as @tool callables |
| strands-sdk-guardrails | Content filtering, safety |
| strands-sdk-model-providers | Bedrock, Anthropic, LiteLLM |
| strands-sdk-metrics | Token usage, latency, observability |
| strands-sdk-prompt-engineering | System prompts, few-shot, CoT |
| strands-sdk-best-practices | Error handling, retries, security |
| strands-sdk-troubleshooting | Common errors + exact fixes |

## Installation

Install the plugin into Claude Code by referencing this repository:

```bash
claude plugin add strands-sdk
```

Or clone and install locally:

```bash
git clone https://github.com/sagarr-shellkode/strands-sdk.git
claude plugin add ./strands-sdk
```

## Usage

### Auto-triggering

Skills activate automatically when Claude Code detects relevant context in your conversation or open files. For example, opening a file that imports `strands` and uses `@tool` will trigger `strands-sdk-tools`.

### `/strands-docs` command

Use the `/strands-docs` command for targeted lookups:

```
/strands-docs how do I add prompt caching to an agent?
/strands-docs show me a fan-out swarm pattern
/strands-docs what is the A2A protocol?
/strands-docs how do I use agents as tools?
/strands-docs BedrockModel constructor parameters
/strands-docs lifecycle hooks before_tool_call
/strands-docs fix AccessDeniedException bedrock
/strands-docs streaming callback example
/strands-docs guardrails configuration
/strands-docs token usage metrics
```

### Example interactions

```
# Get core agent setup
/strands-docs create a basic agent with BedrockModel

# Multi-agent patterns
/strands-docs orchestrate a swarm of 5 parallel agents

# Debugging
/strands-docs why is my tool not being called
```

## License

MIT
