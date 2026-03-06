---
name: strands-sdk-prompt-engineering
description: Use when writing effective system prompts for Strands agents, applying few-shot examples, chain-of-thought reasoning, output formatting instructions, controlling agent persona, or improving response quality and consistency. Triggers on: system prompt, few-shot, chain of thought, prompt quality, output format, agent persona, prompt template.
---

# Prompt Engineering — Strands SDK

## Overview

System prompt quality is the single biggest lever for agent behavior in Strands. Good prompts define role, constraints, output format, and reasoning style. Applied via `Agent(system_prompt=...)`.

## System Prompt Structure

```
1. Role / Persona       — who the agent is
2. Capabilities         — what it can do
3. Constraints          — what it must NOT do
4. Output format        — how to structure responses
5. Reasoning style      — how to think (CoT, step-by-step)
6. Examples (optional)  — few-shot demonstrations
```

## Strong System Prompt Pattern

```python
system_prompt = """You are a senior software architect specializing in distributed systems.

CAPABILITIES:
- Analyze system designs and identify scalability bottlenecks
- Recommend architectural patterns with trade-offs
- Review code for performance and reliability issues

CONSTRAINTS:
- Never recommend technologies you are not confident about
- Always state assumptions explicitly
- Do not write production code — only pseudocode and architecture diagrams

OUTPUT FORMAT:
- Start with a 1-sentence summary
- Use bullet points for lists of more than 3 items
- End with a Key Trade-offs section
- No markdown headers — plain numbered lists only

REASONING:
- Think step by step before giving a final recommendation
- Explicitly state what you do not know
"""

agent = Agent(model=BedrockModel(...), system_prompt=system_prompt)
```

## Few-Shot Examples in System Prompt

```python
system_prompt = """You extract action items from meeting notes.

Format each action item as:
- [OWNER] Task description (due: DATE)

Examples:
Input: "Alice will update the API docs by Friday. Bob needs to review the PR."
Output:
- [Alice] Update API docs (due: Friday)
- [Bob] Review the PR (due: not specified)

Input: "We agreed the team will migrate to PostgreSQL next sprint."
Output:
- [Team] Migrate to PostgreSQL (due: next sprint)
"""
```

## Chain-of-Thought (CoT) Prompting

```python
# In system prompt
system_prompt = "Before answering, think step by step. Show your reasoning, then give the final answer after 'Answer:'"

# Or in the user prompt
result = agent("""
Solve this problem step by step:

A train travels 120km in 1.5 hours. What is its speed?

Think through this step by step before giving your answer.
""")
```

## TTS-Safe Output Format (for Voice Agents)

```python
system_prompt = """You are a podcast host.

OUTPUT RULES (CRITICAL — output is read aloud via text-to-speech):
- NO markdown: no **, *, #, or backticks
- NO lists or bullet points
- Write in natural spoken English only
- 1-2 sentences maximum per response
- Address the speaker by name at the start
"""
```

## Prompt Variables Pattern

```python
def build_system_prompt(role: str, topic: str, style: str) -> str:
    return f"""You are a {role} specializing in {topic}.

Communication style: {style}
Always relate your answers back to {topic}.
"""

agent = Agent(
    model=model,
    system_prompt=build_system_prompt(
        role="AWS solutions architect",
        topic="serverless architectures",
        style="concise and technical"
    )
)
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| System prompt too vague | Add explicit role, constraints, and output format |
| Agent ignores format instructions | Move format rules to end of prompt; use CRITICAL or all-caps |
| Few-shot examples inconsistent | All examples must follow exactly the same format |
| Contradictory instructions | Review for conflicts; last instruction usually wins |
| Prompt too long without caching | Keep under 1024 tokens or use prompt caching (strands-sdk-prompt-caching) |
