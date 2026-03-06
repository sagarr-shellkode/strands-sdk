---
name: strands-sdk-agents-as-tools
description: "Use when wrapping a Strands agent as a callable @tool for another agent, building hierarchical agent systems where an orchestrator delegates to specialist sub-agents, composing agents into a pipeline, or creating dynamic specialist agents on demand. Triggers on: agent as tool, sub-agent, specialist agent, nested agent, delegate to agent, hierarchical agents."
---

# Agents as Tools — Strands SDK

## Overview

Wrap any Strands `Agent` as a `@tool` so an orchestrator agent can call it on demand. The orchestrator decides when to invoke which sub-agent solely from the tool's docstring — just as it decides when to call any other tool. This pattern enables task decomposition, specialization, and multi-level hierarchies without requiring a separate communication protocol.

## Quick Reference

| Concept | Key Point |
|---|---|
| Wrap sub-agent | `@tool` decorator on a function that calls `agent.invoke_async` |
| Orchestrator invocation | Docstring drives when the orchestrator calls this tool |
| State isolation | Each sub-agent has its own `conversation_manager` |
| Clear state | `agent.conversation_manager.clear()` between unrelated tasks |
| Dynamic agents | Create `Agent(...)` inside the tool function for per-call isolation |
| Multi-level | A sub-agent tool can itself have tools that are agents |
| Streaming through | Use `callback_handler` inside the tool to forward tokens upward |
| Error recovery | Catch exceptions inside the tool; return structured error string |
| Caching | Store result in a dict keyed by input hash; return cached on hit |

---

## 1. Basic Pattern

```python
from strands import Agent, tool
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

# Persistent specialist sub-agents (created once, reused)
research_agent = Agent(
    model=model,
    system_prompt="You are a research specialist. Find relevant facts and data. "
                  "Always cite sources or note when information is inferred."
)

writer_agent = Agent(
    model=model,
    system_prompt="You are a technical writer. Write clear, structured documents "
                  "using headings, bullet points, and plain language."
)

@tool
async def research(topic: str) -> str:
    """Research a topic and return key facts, data, and relevant context.

    Use this tool whenever you need factual background before writing,
    analyzing, or making decisions. Do NOT use for tasks that are purely
    about formatting or style.

    Args:
        topic: The specific topic or question to research

    Returns:
        A summary of key facts and data about the topic
    """
    result = await research_agent.invoke_async(f"Research: {topic}")
    return str(result)

@tool
async def write_document(content_brief: str) -> str:
    """Write a structured document from a content brief.

    Use this tool when you have gathered all facts and need to produce
    a formatted, human-readable document or report.

    Args:
        content_brief: Outline and key points the document must cover

    Returns:
        Formatted document with headings and clear structure
    """
    result = await writer_agent.invoke_async(f"Write document: {content_brief}")
    return str(result)

# Orchestrator uses both sub-agents as tools
orchestrator = Agent(
    model=model,
    tools=[research, write_document],
    system_prompt="You orchestrate research and writing tasks. "
                  "First research, then write. Never skip research."
)

result = await orchestrator.invoke_async(
    "Create a technical brief on prompt caching techniques."
)
print(str(result))
```

---

## 2. Tool Docstring Engineering for Reliable Orchestrator Invocation

The orchestrator decides whether to call a tool based entirely on its docstring. A weak docstring causes the orchestrator to skip the tool, call it at the wrong time, or misuse its parameters.

### Docstring Anatomy

```python
@tool
async def analyze_code(code: str, language: str = "python") -> str:
    """Review code for bugs, security issues, and style problems.      # (1) One-line summary: what it does
                                                                        #     Use active verb, be specific.
    Use this tool when:                                                 # (2) Explicit trigger conditions
    - A user asks for a code review                                     #     List concrete scenarios.
    - Code has been written and needs quality checks
    - You need to identify bugs or vulnerabilities before merging
    Do NOT use this tool:                                               # (3) Explicit exclusions
    - For formatting only (use format_code instead)                     #     Prevents wrong-tool invocation.
    - When the task is to generate new code

    Args:                                                               # (4) Args section: name + purpose
        code: The complete source code to review                        #     Include constraints/format hints.
        language: Programming language for syntax-aware analysis        #     Document default behavior.
                  (default: python). Supported: python, javascript,
                  typescript, go, java.

    Returns:                                                            # (5) Returns: describe structure
        JSON string with keys: "bugs" (list), "security_issues"        #     If structured, say so explicitly.
        (list), "style_issues" (list), "overall_score" (0-10).
    """
    ...
```

### Docstring Rules

| Rule | Why It Matters |
|---|---|
| Start with an active verb ("Review", "Analyze", "Generate") | LLM matches verb to user intent |
| Include "Use when" trigger list | Removes ambiguity about when to invoke |
| Include "Do NOT use" exclusions | Prevents calling the wrong specialist |
| Describe return structure explicitly | Orchestrator can parse and act on result correctly |
| Keep one-liner under ~15 words | First line is shown in tool lists; must be scannable |
| Avoid jargon the orchestrator LLM won't recognize | Use plain, unambiguous English |

### Weak vs Strong Docstring

```python
# WEAK — orchestrator often skips this or misuses it
@tool
async def review(input: str) -> str:
    """Review something."""
    ...

# STRONG — orchestrator reliably calls this at the right time
@tool
async def review_pull_request(diff: str) -> str:
    """Analyze a code diff for bugs, security vulnerabilities, and style issues.

    Use this tool when a user submits a pull request or asks for a code review.
    Do NOT use for general questions about code style or language syntax.

    Args:
        diff: The unified diff text (output of `git diff`) to review

    Returns:
        Structured review with sections: Summary, Critical Issues,
        Warnings, Suggestions. Each issue includes line reference and fix.
    """
    ...
```

---

## 3. Passing Context from Orchestrator to Sub-Agent

Sub-agents only see what the tool function passes to them. The orchestrator's broader conversation context is not automatically shared. Inject relevant context explicitly through the tool's input string or a structured prompt.

```python
from dataclasses import dataclass, asdict
import json

@dataclass
class ReviewContext:
    diff: str
    pr_title: str
    author: str
    target_branch: str
    prior_comments: list[str]

@tool
async def review_pull_request(context_json: str) -> str:
    """Analyze a pull request diff given full PR context.

    Args:
        context_json: JSON string with keys: diff, pr_title, author,
                      target_branch, prior_comments (list of strings).

    Returns:
        Structured code review addressing the PR context.
    """
    ctx = json.loads(context_json)
    prompt = f"""Review this pull request.

PR Title: {ctx['pr_title']}
Author: {ctx['author']}
Target branch: {ctx['target_branch']}
Prior review comments: {json.dumps(ctx['prior_comments'], indent=2)}

Diff:
{ctx['diff']}

Focus on blocking issues only. Reference prior comments to avoid repetition.
"""
    result = await review_agent.invoke_async(prompt)
    return str(result)
```

When calling from the orchestrator, serialize context before passing:

```python
# Orchestrator composes the context and calls the tool
ctx = ReviewContext(
    diff=diff_text,
    pr_title="Add rate limiting middleware",
    author="alice",
    target_branch="main",
    prior_comments=["Ensure thread safety in shared state"]
)
result = await orchestrator.invoke_async(
    f"Review this PR: {json.dumps(asdict(ctx))}"
)
```

---

## 4. Sub-Agent Result Formatting (Structured Output for Orchestrator)

Return structured strings (JSON or delimited sections) so the orchestrator can parse, compare, and act on results instead of treating them as opaque text.

```python
import json

@tool
async def analyze_sentiment(texts: str) -> str:
    """Analyze sentiment for a newline-separated list of customer feedback texts.

    Args:
        texts: Newline-separated feedback strings to analyze

    Returns:
        JSON string: {"results": [{"text": str, "sentiment": "positive"|"negative"|"neutral",
        "score": float, "key_themes": list[str]}], "summary": {"positive_pct": float,
        "negative_pct": float, "neutral_pct": float, "top_themes": list[str]}}
    """
    feedback_list = texts.strip().split("\n")
    prompt = f"""Analyze the sentiment of each feedback item. Return ONLY valid JSON
matching this schema exactly:
{{
  "results": [
    {{"text": "...", "sentiment": "positive|negative|neutral",
      "score": 0.0, "key_themes": ["..."]}}
  ],
  "summary": {{
    "positive_pct": 0.0, "negative_pct": 0.0, "neutral_pct": 0.0,
    "top_themes": ["..."]
  }}
}}

Feedback items:
{chr(10).join(f'- {f}' for f in feedback_list)}
"""
    result = await sentiment_agent.invoke_async(prompt)
    raw = str(result)
    # Validate JSON before returning to orchestrator
    try:
        parsed = json.loads(raw)
        return json.dumps(parsed)
    except json.JSONDecodeError:
        # Extract JSON block if agent added surrounding text
        import re
        match = re.search(r'\{.*\}', raw, re.DOTALL)
        if match:
            return match.group(0)
        return json.dumps({"error": "parse_failed", "raw": raw[:500]})
```

The orchestrator can then parse the result and branch on it:

```python
orchestrator = Agent(
    model=model,
    tools=[analyze_sentiment, escalate_to_support, generate_response],
    system_prompt="""You process customer feedback.
After calling analyze_sentiment, parse the JSON result.
If negative_pct > 0.4, call escalate_to_support.
Otherwise call generate_response with the top themes."""
)
```

---

## 5. Multi-Level Hierarchies (Orchestrator → Manager → Worker)

Sub-agents can themselves have tools that are agents, enabling three or more levels. Each level sees only its immediate children.

```python
# Level 3: Worker agents
syntax_checker = Agent(model=model, system_prompt="Check code syntax and type errors only.")
security_scanner = Agent(model=model, system_prompt="Scan code for OWASP Top 10 vulnerabilities only.")
style_linter = Agent(model=model, system_prompt="Check PEP8 style and naming conventions only.")

# Level 2: Manager tools (each wraps a worker)
@tool
async def check_syntax(code: str) -> str:
    """Check code for syntax errors and type annotation issues.

    Args:
        code: Source code to check

    Returns:
        List of syntax/type errors, or "No issues found."
    """
    return str(await syntax_checker.invoke_async(f"Check syntax:\n{code}"))

@tool
async def scan_security(code: str) -> str:
    """Scan code for security vulnerabilities (OWASP Top 10, injection, etc.).

    Args:
        code: Source code to scan

    Returns:
        List of security issues with severity (critical/high/medium/low).
    """
    return str(await security_scanner.invoke_async(f"Security scan:\n{code}"))

@tool
async def lint_style(code: str) -> str:
    """Check code style against PEP8 and project naming conventions.

    Args:
        code: Source code to lint

    Returns:
        List of style violations with line references.
    """
    return str(await style_linter.invoke_async(f"Lint style:\n{code}"))

# Level 2: Manager agent uses worker tools
review_manager = Agent(
    model=model,
    tools=[check_syntax, scan_security, lint_style],
    system_prompt="You are a code review manager. Run all three checks on code "
                  "and consolidate results into a single structured report."
)

# Level 1: Orchestrator tool wraps the manager
@tool
async def full_code_review(code: str, language: str = "python") -> str:
    """Run a comprehensive code review covering syntax, security, and style.

    Use this tool when a complete multi-dimensional code review is needed
    before merging or releasing code. Runs syntax, security, and style
    checks in sequence via specialist sub-agents.

    Args:
        code: Complete source code to review
        language: Language name for context (default: python)

    Returns:
        Consolidated review report with sections for syntax, security,
        style issues, and an overall pass/fail recommendation.
    """
    result = await review_manager.invoke_async(
        f"Review this {language} code:\n{code}"
    )
    return str(result)

# Level 0: Top-level orchestrator
orchestrator = Agent(
    model=model,
    tools=[full_code_review, apply_fixes, open_pr],
    system_prompt="You manage the code review and release pipeline."
)
```

### Hierarchy Depth Guidelines

| Levels | When to Use |
|---|---|
| 2 (orchestrator + workers) | Most use cases; lowest latency |
| 3 (orchestrator + manager + workers) | Workers need coordination logic |
| 4+ | Rarely justified; adds latency and debugging complexity |

---

## 6. Stateful vs Stateless Sub-Agents

### Stateless Sub-Agent (new agent per call)

Each invocation creates a fresh `Agent` instance. No conversation history bleeds between calls. Use for pure functions or when isolation is critical.

```python
@tool
async def classify_document(text: str, categories: str) -> str:
    """Classify a document into one of the provided categories.

    Use for single-document classification. Each call is independent.

    Args:
        text: Document text (up to 5000 words)
        categories: Comma-separated list of valid categories

    Returns:
        JSON: {"category": str, "confidence": float, "reasoning": str}
    """
    # Fresh agent per call — no state leakage between documents
    classifier = Agent(
        model=model,
        system_prompt=(
            f"You classify documents into exactly one of these categories: "
            f"{categories}. Return only valid JSON."
        )
    )
    result = await classifier.invoke_async(f"Classify:\n{text}")
    return str(result)
```

### Stateful Sub-Agent (persistent across calls)

A single agent instance is reused so it accumulates context across multiple orchestrator turns. Use when the sub-agent needs memory of prior calls within the same session.

```python
# Module-level: persists for the lifetime of the process
interview_agent = Agent(
    model=model,
    system_prompt="You are conducting a structured technical interview. "
                  "Remember all previous questions and answers to avoid repetition "
                  "and build follow-up questions from prior responses."
)

@tool
async def ask_interview_question(candidate_answer: str) -> str:
    """Continue a technical interview by asking the next question.

    The interview agent remembers all prior questions and answers.
    Pass the candidate's latest response; the agent will generate
    the next contextually appropriate follow-up question.

    Args:
        candidate_answer: The candidate's response to the previous question

    Returns:
        The next interview question to ask the candidate.
    """
    result = await interview_agent.invoke_async(candidate_answer)
    return str(result)

def reset_interview():
    """Call between interview sessions to clear accumulated history."""
    interview_agent.conversation_manager.clear()
```

### Choosing Between Stateful and Stateless

| Criterion | Stateless (new per call) | Stateful (reuse instance) |
|---|---|---|
| Isolation between calls | Complete | None — history accumulates |
| Memory of prior calls | Not available | Available |
| Thread safety | Safe (no shared state) | Requires care in concurrent contexts |
| Latency | Slightly higher (init cost) | Lower after first call |
| Use for | Classification, translation, extraction | Interviews, tutoring, iterative refinement |

---

## 7. Sub-Agent with Its Own Tools

A sub-agent can have its own tools (HTTP, file I/O, databases, other agents). The orchestrator does not need to know about them.

```python
from strands.tools import http_request, file_read

# Sub-agent with real-world tools
data_agent = Agent(
    model=model,
    tools=[http_request, file_read],
    system_prompt="You fetch and analyze data. Use http_request for live APIs "
                  "and file_read for local datasets. Summarize findings concisely."
)

@tool
async def fetch_and_analyze(data_source: str, question: str) -> str:
    """Fetch data from a URL or local file and answer a specific question about it.

    The sub-agent can make HTTP requests and read files autonomously.
    Use when the question requires live or file-based data.

    Args:
        data_source: A URL (https://...) or absolute file path (/path/to/file)
        question: The specific question to answer using the fetched data

    Returns:
        Answer to the question based on retrieved data, with source cited.
    """
    prompt = (
        f"Data source: {data_source}\n"
        f"Question: {question}\n"
        f"Fetch the data and answer the question. Cite the source."
    )
    result = await data_agent.invoke_async(prompt)
    return str(result)
```

A sub-agent with MCP tools:

```python
from strands.tools.mcp import MCPClient

async def build_db_agent():
    async with MCPClient({
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-postgres", DATABASE_URL]
    }) as mcp:
        db_tools = await mcp.list_tools_async()
        db_agent = Agent(
            model=model,
            tools=db_tools,
            system_prompt="You query a PostgreSQL database. Write safe, read-only SQL."
        )
        # Use db_agent within this context manager scope
        return db_agent
```

---

## 8. Streaming Results from Sub-Agent Back Through Tool

By default, tool return values are complete strings — the orchestrator sees nothing until the sub-agent finishes. To surface partial results, use a `callback_handler` inside the tool and write tokens to a shared queue or call an external sink.

### Pattern A: Collect and Forward via Callback

```python
import asyncio

@tool
async def stream_analysis(document: str) -> str:
    """Analyze a long document and stream findings as they are generated.

    Use for large document analysis where partial results are valuable
    before the full analysis completes.

    Args:
        document: Full text of the document to analyze

    Returns:
        Complete analysis result (streaming tokens are also forwarded
        to the live_output queue during processing).
    """
    tokens: list[str] = []

    def collect_token(event: dict):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                tokens.append(text)
                # Forward token to an external queue for SSE/WebSocket
                live_output_queue.put_nowait(text)

    streaming_agent = Agent(
        model=model,
        callback_handler=collect_token,
        system_prompt="You analyze documents section by section."
    )
    await streaming_agent.invoke_async(f"Analyze:\n{document}")
    return "".join(tokens)
```

### Pattern B: Async Generator Tool (manual orchestration)

When you need true streaming at the orchestrator level, manage the pipeline manually rather than through the orchestrator's tool-call mechanism:

```python
async def streaming_pipeline(document: str):
    """Manual pipeline that yields tokens from sub-agent as they arrive."""
    queue: asyncio.Queue[str | None] = asyncio.Queue()

    def on_token(event: dict):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                queue.put_nowait(text)

    agent = Agent(model=model, callback_handler=on_token)

    # Run agent invocation concurrently
    async def run():
        await agent.invoke_async(f"Analyze:\n{document}")
        queue.put_nowait(None)  # sentinel

    asyncio.create_task(run())

    # Yield tokens as they arrive
    while True:
        token = await queue.get()
        if token is None:
            break
        yield token
```

---

## 9. Error Handling (Sub-Agent Fails, Orchestrator Recovers)

Never let a sub-agent exception propagate raw to the orchestrator. Catch it inside the tool and return a structured error string so the orchestrator can decide how to recover.

```python
import logging
from botocore.exceptions import ClientError

logger = logging.getLogger(__name__)

@tool
async def summarize_document(text: str) -> str:
    """Summarize a document into three key points.

    Args:
        text: Document text to summarize (up to 10,000 words)

    Returns:
        Three-bullet summary, OR a JSON error object if processing failed.
        Error format: {"error": "error_type", "message": str, "recoverable": bool}
    """
    if not text or not text.strip():
        return '{"error": "empty_input", "message": "No text provided", "recoverable": false}'

    if len(text) > 50_000:
        return (
            '{"error": "input_too_long", '
            '"message": "Document exceeds 50,000 character limit. Split into smaller chunks.", '
            '"recoverable": true}'
        )

    try:
        result = await summarizer_agent.invoke_async(f"Summarize:\n{text[:50_000]}")
        return str(result)

    except ClientError as e:
        code = e.response["Error"]["Code"]
        if code == "ThrottlingException":
            logger.warning("summarize_document: throttled by Bedrock")
            return (
                '{"error": "throttled", '
                '"message": "Service temporarily busy. Retry in 30 seconds.", '
                '"recoverable": true}'
            )
        elif code == "AccessDeniedException":
            logger.error("summarize_document: IAM permission denied")
            return (
                '{"error": "permission_denied", '
                '"message": "Agent lacks Bedrock InvokeModel permission.", '
                '"recoverable": false}'
            )
        else:
            logger.error(f"summarize_document: AWS error {code}", exc_info=True)
            return f'{{"error": "aws_error", "message": "{code}", "recoverable": false}}'

    except Exception as e:
        logger.error(f"summarize_document: unexpected error: {e}", exc_info=True)
        return (
            '{"error": "unexpected", '
            f'"message": "{str(e)[:200]}", '
            '"recoverable": false}'
        )
```

Configure the orchestrator to handle error returns:

```python
orchestrator = Agent(
    model=model,
    tools=[summarize_document, fetch_document, store_summary],
    system_prompt="""You summarize documents.
If summarize_document returns a JSON object with "error" key:
- If "recoverable" is true, retry once after waiting.
- If "recoverable" is false, report the error to the user and stop.
Never proceed to store_summary if summarize_document returned an error."""
)
```

---

## 10. Sub-Agent Caching (Same Input → Cached Result)

Cache sub-agent results keyed by a hash of the input. Useful when the orchestrator may call the same tool with identical inputs multiple times in one session (e.g., repeated research on the same topic).

```python
import hashlib
import json
from functools import lru_cache

# Module-level in-memory cache
_research_cache: dict[str, str] = {}

@tool
async def research_cached(topic: str) -> str:
    """Research a topic, using cached results for previously seen topics.

    Identical topic strings return cached results immediately without
    calling the LLM. Cache persists for the lifetime of the process.

    Args:
        topic: Topic to research (used verbatim as cache key)

    Returns:
        Research findings. Includes "[CACHED]" prefix if served from cache.
    """
    cache_key = hashlib.sha256(topic.strip().lower().encode()).hexdigest()

    if cache_key in _research_cache:
        cached = _research_cache[cache_key]
        return f"[CACHED] {cached}"

    result = str(await research_agent.invoke_async(f"Research: {topic}"))
    _research_cache[cache_key] = result
    return result

def clear_research_cache():
    """Invalidate all cached research results."""
    _research_cache.clear()
```

For TTL-based expiry:

```python
import time

_cache_with_ttl: dict[str, tuple[str, float]] = {}
CACHE_TTL_SECONDS = 3600  # 1 hour

@tool
async def research_ttl(topic: str) -> str:
    """Research a topic with 1-hour result caching.

    Args:
        topic: Topic to research

    Returns:
        Research findings (from cache if result is under 1 hour old).
    """
    key = hashlib.sha256(topic.strip().lower().encode()).hexdigest()
    now = time.monotonic()

    if key in _cache_with_ttl:
        value, timestamp = _cache_with_ttl[key]
        if now - timestamp < CACHE_TTL_SECONDS:
            return f"[CACHED] {value}"

    result = str(await research_agent.invoke_async(f"Research: {topic}"))
    _cache_with_ttl[key] = (result, now)
    return result
```

---

## 11. Dynamic Sub-Agent Creation

Create `Agent` instances inside the tool function when each call requires a different configuration (different system prompt, tools, or model).

```python
@tool
async def analyze_with_persona(persona: str, question: str) -> str:
    """Analyze a question from a specific named expert persona.

    Use when the same question needs to be viewed through a specialized
    lens (e.g., security engineer, product manager, lawyer).

    Args:
        persona: Expert role description, e.g. "senior security engineer",
                 "startup CFO", "GDPR compliance lawyer"
        question: The question or problem to analyze from that perspective

    Returns:
        Analysis written from the stated expert's point of view.
    """
    specialist = Agent(
        model=model,
        system_prompt=(
            f"You are a {persona}. Answer every question strictly from the "
            f"perspective of your role and professional expertise. "
            f"Acknowledge tradeoffs from your vantage point."
        )
    )
    result = await specialist.invoke_async(question)
    return str(result)
```

Dynamic creation with different models based on task complexity:

```python
haiku_model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)
sonnet_model = BedrockModel(
    model_id="anthropic.claude-sonnet-4-5-20251001-v1:0",
    region_name="us-east-1"
)

@tool
async def solve_problem(problem: str, complexity: str = "simple") -> str:
    """Solve a problem using a model matched to its complexity.

    Args:
        problem: Problem description or question
        complexity: "simple" (fast, cheap) or "complex" (thorough, slower)

    Returns:
        Solution with reasoning appropriate to the complexity level.
    """
    chosen_model = haiku_model if complexity == "simple" else sonnet_model
    solver = Agent(model=chosen_model, system_prompt="Solve problems step by step.")
    return str(await solver.invoke_async(problem))
```

---

## 12. Testing Hierarchical Agents

### Unit Test: Test the Tool Function in Isolation

Mock the sub-agent to test the tool's own logic (input validation, output parsing, error handling) without making real LLM calls.

```python
import pytest
from unittest.mock import AsyncMock, patch, MagicMock

@pytest.mark.asyncio
async def test_summarize_document_empty_input():
    """Tool should return structured error for empty input, not raise."""
    import json
    result = await summarize_document("")
    parsed = json.loads(result)
    assert parsed["error"] == "empty_input"
    assert parsed["recoverable"] is False

@pytest.mark.asyncio
async def test_summarize_document_success():
    """Tool should return sub-agent output as-is on success."""
    mock_result = MagicMock()
    mock_result.__str__ = lambda self: "Point 1. Point 2. Point 3."

    with patch.object(summarizer_agent, "invoke_async", new=AsyncMock(return_value=mock_result)):
        result = await summarize_document("This is a test document.")
    assert "Point 1" in result

@pytest.mark.asyncio
async def test_summarize_document_throttle():
    """Tool should return recoverable error JSON on ThrottlingException."""
    import json
    from botocore.exceptions import ClientError

    throttle_error = ClientError(
        {"Error": {"Code": "ThrottlingException", "Message": "Rate exceeded"}},
        "InvokeModel"
    )
    with patch.object(summarizer_agent, "invoke_async", new=AsyncMock(side_effect=throttle_error)):
        result = await summarize_document("Some text")
    parsed = json.loads(result)
    assert parsed["error"] == "throttled"
    assert parsed["recoverable"] is True
```

### Integration Test: Test the Full Hierarchy

```python
@pytest.mark.asyncio
@pytest.mark.integration  # mark to run only against real AWS
async def test_research_write_pipeline():
    """Orchestrator should call research then write_document in sequence."""
    calls: list[str] = []

    original_research = research_agent.invoke_async
    original_writer = writer_agent.invoke_async

    async def tracked_research(prompt):
        calls.append("research")
        return await original_research(prompt)

    async def tracked_writer(prompt):
        calls.append("write")
        return await original_writer(prompt)

    with (
        patch.object(research_agent, "invoke_async", side_effect=tracked_research),
        patch.object(writer_agent, "invoke_async", side_effect=tracked_writer),
    ):
        result = await orchestrator.invoke_async(
            "Write a brief on Python async best practices."
        )

    assert "research" in calls, "Orchestrator must call research sub-agent"
    assert "write" in calls, "Orchestrator must call writer sub-agent"
    assert calls.index("research") < calls.index("write"), "Research must precede writing"
    assert len(str(result)) > 100
```

### Test State Isolation

```python
@pytest.mark.asyncio
async def test_stateless_tool_no_bleed():
    """Each call to a stateless tool must be independent."""
    mock_result = MagicMock()
    mock_result.__str__ = lambda self: "Category: A"

    with patch("mymodule.Agent") as MockAgent:
        instance = AsyncMock()
        instance.invoke_async.return_value = mock_result
        MockAgent.return_value = instance

        await classify_document("text one", "A,B,C")
        await classify_document("text two", "A,B,C")

    # Agent constructor called twice — new instance per call
    assert MockAgent.call_count == 2
```

---

## 13. Real-World Example: Code Review Pipeline

A three-level hierarchy: orchestrator → review manager → specialist reviewers.

```python
import json
from strands import Agent, tool
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

# Workers
_security_agent = Agent(model=model, system_prompt="You are an application security expert. Find OWASP Top 10 vulnerabilities, injection flaws, and authentication issues in code.")
_perf_agent    = Agent(model=model, system_prompt="You are a performance engineer. Identify O(n^2) algorithms, N+1 queries, memory leaks, and blocking I/O in code.")
_style_agent   = Agent(model=model, system_prompt="You are a code style reviewer. Check PEP8, naming conventions, missing docstrings, and overly complex functions.")

@tool
async def security_review(code: str) -> str:
    """Scan code for security vulnerabilities (OWASP Top 10, injections, auth issues).

    Args:
        code: Source code to scan

    Returns:
        JSON list of findings: [{"severity": "critical|high|medium|low",
        "issue": str, "line_hint": str, "fix": str}]
    """
    result = await _security_agent.invoke_async(
        f"Security scan — return only a JSON array of findings:\n{code}"
    )
    return str(result)

@tool
async def performance_review(code: str) -> str:
    """Identify performance issues: O(n^2) algorithms, N+1 queries, memory leaks.

    Args:
        code: Source code to analyze

    Returns:
        JSON list of performance findings with severity and suggested fix.
    """
    result = await _perf_agent.invoke_async(
        f"Performance review — return only a JSON array of findings:\n{code}"
    )
    return str(result)

@tool
async def style_review(code: str) -> str:
    """Check code style: PEP8, naming, docstrings, complexity.

    Args:
        code: Source code to lint

    Returns:
        JSON list of style issues with line references and corrections.
    """
    result = await _style_agent.invoke_async(
        f"Style review — return only a JSON array of findings:\n{code}"
    )
    return str(result)

# Manager: runs all three workers, consolidates
_review_manager = Agent(
    model=model,
    tools=[security_review, performance_review, style_review],
    system_prompt=(
        "You are a code review manager. When given code, always run all three "
        "reviews: security_review, performance_review, style_review. "
        "Consolidate results into a report with sections: CRITICAL, HIGH, MEDIUM, LOW, STYLE. "
        "Include an overall PASS/FAIL verdict: fail if any critical or high severity issues exist."
    )
)

@tool
async def full_code_review(code: str, pr_title: str = "") -> str:
    """Run a comprehensive code review: security, performance, and style analysis.

    Use before merging any pull request. Runs all three review dimensions
    via specialist sub-agents and produces a consolidated, actionable report.

    Args:
        code: Complete source code or diff to review
        pr_title: Optional PR title for context

    Returns:
        Consolidated review report with PASS/FAIL verdict and prioritized issues.
    """
    context = f"PR: {pr_title}\n\n" if pr_title else ""
    try:
        result = await _review_manager.invoke_async(f"{context}Review this code:\n{code}")
        return str(result)
    except Exception as e:
        logger.error(f"full_code_review failed: {e}", exc_info=True)
        return f'{{"error": "review_failed", "message": "{str(e)[:200]}"}}'

# Orchestrator: manages the PR lifecycle
pr_orchestrator = Agent(
    model=model,
    tools=[full_code_review],
    system_prompt=(
        "You manage the pull request review process. "
        "When given a PR diff, call full_code_review. "
        "If the result contains FAIL verdict, respond with the critical issues. "
        "If PASS, confirm approval."
    )
)
```

---

## 14. Real-World Example: Research + Write Pipeline

```python
import json
from strands import Agent, tool
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

_researcher = Agent(
    model=model,
    system_prompt=(
        "You are a research analyst. For any topic, produce a structured summary with: "
        "1) Background, 2) Key concepts, 3) Current state of the art, "
        "4) Open questions. Use bullet points. Cite uncertainty explicitly."
    )
)

_writer = Agent(
    model=model,
    system_prompt=(
        "You are a technical writer. Transform research briefs into polished documents "
        "for a technical audience. Use clear headings, active voice, and concrete examples. "
        "Do not add facts not present in the brief."
    )
)

_editor = Agent(
    model=model,
    system_prompt=(
        "You are a senior editor. Review documents for clarity, consistency, redundancy, "
        "and factual accuracy. Return the edited document only — no commentary."
    )
)

_research_cache: dict[str, str] = {}

@tool
async def research_topic(topic: str) -> str:
    """Research a topic deeply and return a structured findings brief.

    Use this FIRST before writing anything. Do not call write_article
    without first calling this tool.

    Args:
        topic: Topic, question, or technology to research in depth

    Returns:
        Structured research brief with background, key concepts,
        current state of the art, and open questions.
    """
    key = topic.strip().lower()
    if key in _research_cache:
        return f"[CACHED] {_research_cache[key]}"
    result = str(await _researcher.invoke_async(f"Research: {topic}"))
    _research_cache[key] = result
    return result

@tool
async def write_article(research_brief: str, target_length: str = "medium") -> str:
    """Write a polished technical article from a research brief.

    Call this AFTER research_topic. Pass the full research brief as input.
    Do NOT generate facts — only transform the brief into article form.

    Args:
        research_brief: Full text output from research_topic
        target_length: "short" (~300 words), "medium" (~600 words),
                       "long" (~1200 words). Default: medium.

    Returns:
        Formatted technical article with title, introduction, body sections,
        and conclusion.
    """
    prompt = (
        f"Write a {target_length} technical article based on this research:\n\n"
        f"{research_brief}\n\nUse headings and concrete examples."
    )
    return str(await _writer.invoke_async(prompt))

@tool
async def edit_article(draft: str) -> str:
    """Edit a draft article for clarity, flow, and conciseness.

    Call this AFTER write_article to produce the final polished version.

    Args:
        draft: Draft article text from write_article

    Returns:
        Edited final article — same structure, improved language.
    """
    return str(await _editor.invoke_async(f"Edit this article:\n{draft}"))

# Orchestrator ties the pipeline together
content_orchestrator = Agent(
    model=model,
    tools=[research_topic, write_article, edit_article],
    system_prompt=(
        "You produce technical articles in three mandatory steps: "
        "1. Call research_topic with the subject. "
        "2. Call write_article with the full research output. "
        "3. Call edit_article with the full draft output. "
        "Return the final edited article to the user. "
        "Never skip a step."
    )
)
```

---

## 15. Real-World Example: Data Analysis Chain

```python
import json
from strands import Agent, tool
from strands.tools import file_read, http_request
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-sonnet-4-5-20251001-v1:0",
    region_name="us-east-1"
)

_ingestor = Agent(
    model=model,
    tools=[file_read, http_request],
    system_prompt=(
        "You ingest raw data from files or APIs. "
        "Return a JSON object: {\"rows\": [...], \"columns\": [...], \"row_count\": int, "
        "\"sample\": [first 3 rows], \"source\": str}"
    )
)

_analyzer = Agent(
    model=model,
    system_prompt=(
        "You analyze structured datasets. Compute descriptive statistics, "
        "identify trends, outliers, and correlations. Return findings as JSON."
    )
)

_reporter = Agent(
    model=model,
    system_prompt=(
        "You write executive data reports. Transform JSON analysis findings "
        "into a clear narrative with: Key Finding, Supporting Data, "
        "Recommended Actions. Use plain language."
    )
)

@tool
async def ingest_data(source: str) -> str:
    """Load and parse data from a file path or HTTP URL.

    Use as the first step in any data analysis workflow.
    Supports CSV, JSON, and plain-text formats.

    Args:
        source: Absolute file path (/data/sales.csv) or URL (https://...)

    Returns:
        JSON with keys: rows (list), columns (list), row_count (int),
        sample (first 3 rows), source (str).
    """
    try:
        result = str(await _ingestor.invoke_async(f"Ingest data from: {source}"))
        # Validate it's parseable JSON before returning
        json.loads(result)
        return result
    except json.JSONDecodeError:
        return json.dumps({"error": "parse_failed", "raw_preview": result[:300]})
    except Exception as e:
        return json.dumps({"error": str(e)})

@tool
async def analyze_dataset(dataset_json: str, analysis_goals: str) -> str:
    """Analyze a dataset for patterns, trends, outliers, and correlations.

    Call after ingest_data. Pass the full JSON output from ingest_data.

    Args:
        dataset_json: JSON string from ingest_data
        analysis_goals: What to look for, e.g. "monthly revenue trends and
                        top-performing SKUs"

    Returns:
        JSON analysis: {"summary_stats": {...}, "trends": [...],
        "outliers": [...], "correlations": [...], "key_finding": str}
    """
    prompt = (
        f"Analyze this dataset for: {analysis_goals}\n\n"
        f"Dataset:\n{dataset_json}\n\n"
        f"Return only valid JSON with keys: summary_stats, trends, outliers, "
        f"correlations, key_finding."
    )
    result = str(await _analyzer.invoke_async(prompt))
    try:
        json.loads(result)
        return result
    except json.JSONDecodeError:
        import re
        match = re.search(r'\{.*\}', result, re.DOTALL)
        return match.group(0) if match else json.dumps({"error": "parse_failed"})

@tool
async def generate_report(analysis_json: str, audience: str = "executive") -> str:
    """Generate a narrative report from analysis findings.

    Call after analyze_dataset. Pass the full JSON output from analyze_dataset.

    Args:
        analysis_json: JSON string from analyze_dataset
        audience: "executive" (plain language, focus on decisions) or
                  "technical" (include statistical details)

    Returns:
        Formatted report with: Key Finding, Supporting Data, Recommended Actions.
    """
    prompt = (
        f"Write a {audience}-level data report from these findings:\n{analysis_json}"
    )
    return str(await _reporter.invoke_async(prompt))

# Orchestrator
data_orchestrator = Agent(
    model=model,
    tools=[ingest_data, analyze_dataset, generate_report],
    system_prompt=(
        "You run data analysis pipelines. Always follow this exact sequence: "
        "1. ingest_data → 2. analyze_dataset → 3. generate_report. "
        "Pass the full output of each step as input to the next. "
        "If any step returns an error key, report it and stop."
    )
)
```

---

## 16. State Isolation

Each sub-agent maintains its own conversation history. The orchestrator sees only the tool return value, not the sub-agent's internal conversation.

```python
# Clear sub-agent state between unrelated tasks
research_agent.conversation_manager.clear()
writer_agent.conversation_manager.clear()
```

Clear all sub-agents in a session teardown:

```python
def reset_session():
    """Clear all sub-agent conversation state. Call between user sessions."""
    for agent in [research_agent, writer_agent, review_manager]:
        agent.conversation_manager.clear()
    _research_cache.clear()
```

---

## 17. Expanded Troubleshooting

| Symptom | Root Cause | Fix |
|---|---|---|
| Orchestrator never calls the sub-agent tool | Weak or missing docstring | Add explicit "Use when" trigger conditions; use active verbs |
| Orchestrator calls wrong sub-agent | Overlapping docstrings | Add "Do NOT use" exclusions to differentiate tools |
| Sub-agent result ignored by orchestrator | Return format is unparseable prose | Return structured JSON; document schema in Returns section |
| Orchestrator calls tool with wrong argument | Ambiguous arg description | Add type, format, and example to each Args entry |
| Sub-agent has old context from prior call | Shared stateful agent not cleared | Call `agent.conversation_manager.clear()` between sessions |
| Tool raises exception, orchestrator halts | Exception escaped from tool | Wrap all agent calls in try/except; return error JSON |
| Tool always returns `[CACHED]` stale data | Cache not invalidated | Call `clear_research_cache()` or add TTL; key on content hash |
| Three-level hierarchy times out | Too many sequential LLM calls | Parallelize independent worker calls with `asyncio.gather` |
| Sub-agent makes wrong tool calls internally | Sub-agent system prompt too broad | Narrow system prompt to single responsibility; restrict tools list |
| Orchestrator skips intermediate steps | Step sequence not enforced in prompt | Number steps explicitly; add "never skip" instruction |
| Sub-agent returns truncated result | Input too long for context window | Truncate or chunk input before passing; note limit in docstring |
| `invoke_async` called in sync context | Mixing sync/async | Use `asyncio.run()` or ensure the call is inside an async function |
| Dynamic agent creation causes high latency | Agent instantiated on every call | Cache agent instances for fixed personas; create dynamically only when config varies |
| JSON parse fails on sub-agent output | Agent prefixed with commentary text | Use `re.search(r'\{.*\}', result, re.DOTALL)` to extract JSON block |
| Worker agents disagree on facts | Each worker has independent context | Have a manager agent synthesize and resolve conflicts |

---

## 18. Common Mistakes

| Mistake | Fix |
|---|---|
| Sub-agent tool has no docstring | Add docstring — orchestrator uses it to decide when to call |
| One-line docstring with no trigger conditions | Add "Use when" and "Do NOT use" sections |
| Return type not described | Document return structure; use JSON for structured data |
| Sub-agent shares state across unrelated calls | Call `conversation_manager.clear()` between sessions |
| Orchestrator never calls sub-agent | Rephrase docstring to explicitly name the trigger scenario |
| Sync sub-agent call in async tool | Use `await agent.invoke_async(...)` not `agent(...)` |
| Exception propagates from tool to orchestrator | Catch all exceptions inside tool; return error JSON |
| All workers called sequentially when independent | Use `asyncio.gather` to parallelize independent sub-agent calls |
| Manager sub-agent has too many tools | Each manager should have 2–5 tightly-scoped tools maximum |
| Dynamic agent created with no system prompt | Always set `system_prompt` — an agent without one behaves unpredictably |
