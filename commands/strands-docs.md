---
description: Look up Strands-agents SDK documentation for a specific topic
argument-hint: <topic>
allowed-tools: [Read, Glob]
---

# Strands SDK Docs

The user asked for Strands SDK documentation on: **$ARGUMENTS**

## Instructions

1. Match the topic argument to the most relevant skill(s) from this table:

| Argument keywords | Skill to surface |
|---|---|
| core, agent, basic, setup, install | strands-sdk-core |
| tool, tools, @tool, mcp, built-in | strands-sdk-tools |
| cache, caching, cache_point, cost | strands-sdk-prompt-caching |
| conversation, history, multi-turn, memory | strands-sdk-conversation |
| stream, streaming, async, callback | strands-sdk-streaming |
| hook, hooks, lifecycle, middleware | strands-sdk-hooks |
| multi-agent, multiagent, orchestrat | strands-sdk-multiagent |
| a2a, agent-to-agent, inter-agent | strands-sdk-a2a |
| swarm, swarms, parallel, fan-out | strands-sdk-agent-swarms |
| agent-as-tool, sub-agent, delegate | strands-sdk-agents-as-tools |
| guardrail, safety, filter, pii | strands-sdk-guardrails |
| provider, model, bedrock, litellm, anthropic | strands-sdk-model-providers |
| metric, metrics, token, latency, cost, otel | strands-sdk-metrics |
| prompt, system-prompt, few-shot, cot | strands-sdk-prompt-engineering |
| best-practice, production, retry, security | strands-sdk-best-practices |
| error, debug, troubleshoot, fix, exception | strands-sdk-troubleshooting |

2. Read the matching SKILL.md file from `skills/<skill-name>/SKILL.md` in this plugin directory.

3. Present the content clearly to the user with:
   - The relevant sections for their query
   - Code examples if applicable
   - A note pointing to related skills if the topic spans multiple areas

4. If the topic does not match any skill, list all available topics and ask the user to clarify.
