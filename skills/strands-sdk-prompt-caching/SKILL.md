---
name: strands-sdk-prompt-caching
description: "Use when reducing token costs with prompt caching, using cache_point in messages, setting up caching with Bedrock or Anthropic models, checking cache hit metrics, debugging why caching is not working, caching tool definitions, warming the cache before requests, structuring prompts for maximum cache hits, or applying production cache-aware agent patterns. Triggers on: cache_point, prompt cache, CacheConfig, reduce cost, cache hit, cache_read_input_tokens, cache warming, cache invalidation, cache miss, cache-aware agent."
---

# Prompt Caching — Strands SDK

## Overview

Prompt caching lets you cache frequently-repeated context — system prompts, large documents, tool definitions — so subsequent requests only pay for the portion of input that is not already in cache. On a warm cache hit, those cached tokens are billed at roughly 10% of the normal input token rate (Anthropic API) or the equivalent Bedrock discount. For agents that repeat the same large context across many calls, this routinely cuts input-token costs by 70–90%.

Caching is supported on:
- **AWS Bedrock** — Claude 3 Haiku, Claude 3.5 Sonnet/Haiku, Claude 3.7 Sonnet and later via `BedrockModel`
- **Anthropic API** — same model family via `AnthropicModel`

---

## Quick Reference

| Concept | Detail |
|---|---|
| Mark cache boundary | `{"cache_point": {"type": "default"}}` as last item in a content array |
| System prompt cache | Strands auto-adds cache_point for system prompts above the threshold |
| Minimum block size | Content before a cache_point must be **>1024 tokens** |
| Cache TTL | **5 minutes** (Anthropic API); Bedrock may differ by model/region |
| Cache scope | Per-model, per-account — never shared across accounts or users |
| Check cache hit | `cache_read_input_tokens` > 0 in usage metadata |
| Check cache write | `cache_creation_input_tokens` > 0 (first call, populating cache) |
| Cacheable content | System prompt, user message content blocks, tool definitions |

---

## How Prompt Caching Works

When you mark a cache boundary with `cache_point`, the model hashes the exact sequence of tokens that appear before that marker. On the first request, those tokens are processed normally and written into a server-side cache — you pay full input-token price plus a small cache-write surcharge. On every subsequent request within the TTL that presents the same token sequence up to that boundary, the server skips re-processing those tokens and serves them from cache. You pay only the discounted cache-read rate for those tokens, and full price only for the uncached portion that follows.

**What gets cached:**
- Text content blocks in the system prompt
- Text/image content blocks in user messages up to a cache_point
- Tool definition JSON schemas listed before a cache_point

**What is NOT cached:**
- Content appearing after the last cache_point in a request
- The assistant's response
- Tool call results (they appear after the cache boundary)
- Anything below the 1024-token minimum

**Cache key:** The exact bytes of every token in sequence from the start of the request up to the cache_point. A single-character difference anywhere in that prefix produces a full cache miss.

---

## Cost Calculation Examples

### Example 1 — Repeated system prompt (typical agent loop)

Scenario: 8,000-token system prompt, 200-token user question, 500-token answer. 1,000 requests/day. Claude 3.5 Haiku pricing (approximate illustrative figures).

| Token type | Price (per 1M) | Without cache | With cache (after 1st call) |
|---|---|---|---|
| Input (normal) | $1.00 | 8,200 tokens × 1,000 = 8.2B tokens → **$8.20/day** | 200 tokens × 999 = ~0.2B → **$0.20/day** |
| Cache write | $1.25 | — | 8,000 tokens × 1 call → **$0.01** (one-time) |
| Cache read | $0.10 | — | 8,000 tokens × 999 calls = ~8B → **$0.80/day** |
| Output | $5.00 | 500 × 1,000 = 0.5B → $2.50/day | same → $2.50/day |
| **Total input cost** | | **$8.20/day** | **~$1.01/day** |

**Savings: ~88% on input tokens.**

### Example 2 — Document Q&A (same 50k-token document, 10 questions)

- Without cache: 50,000 × 10 = 500,000 input tokens billed at full rate
- With cache: 50,000 written once (cache-write rate) + 50,000 × 9 at cache-read rate (10% of normal)
- Net input-token cost reduction: approximately **82%** across those 10 calls

### Example 3 — Small prompt (cache miss scenario)

If your system prompt is only 800 tokens, no cache_point is effective — you must surpass the 1024-token minimum. In this case caching provides **0% savings**; the cache_point is silently ignored.

---

## Caching the System Prompt

Strands automatically appends a cache_point to the system prompt when it detects the content is long enough. You do not need to add it manually for system prompts.

```python
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

# Large system prompt (must exceed 1024 tokens for cache to activate)
system_prompt = """
You are an expert technical assistant for a financial services platform.

PLATFORM CONTEXT
----------------
[...10,000 words of product documentation, policy rules, and compliance guidelines...]
""".strip()

agent = Agent(model=model, system_prompt=system_prompt)

# First call — pays full input price + cache write surcharge
result1 = agent("What is the refund policy for premium accounts?")

# Second call — system prompt served from cache at ~10% cost
result2 = agent("What are the KYC requirements?")
```

---

## Manual cache_point in User Messages

Place a `cache_point` block as the last element of a content array to cache everything in that array up to that marker.

```python
from strands import Agent
from strands.models import BedrockModel

agent = Agent(model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0"))

large_document = "..." * 2000  # must be >1024 tokens

messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": f"Reference document:\n\n{large_document}"
            },
            # Everything above this line is cached after the first call
            {"cache_point": {"type": "default"}}
        ]
    },
    {
        "role": "assistant",
        "content": "I have read the document. What would you like to know?"
    },
    {
        "role": "user",
        "content": "Summarize the executive summary section."  # only this is billed at full rate
    }
]

result = agent("", messages=messages)
```

---

## Caching Tool Definitions

Tool definitions (JSON schemas) are sent with every request and can be large — especially when you register many tools. Cache them alongside the system prompt by placing a cache_point in the tool list or immediately after the system prompt block.

```python
from strands import Agent, tool
from strands.models import BedrockModel

@tool
def search_knowledge_base(query: str, top_k: int = 5) -> list[dict]:
    """Search internal knowledge base. Returns top_k most relevant documents.

    Args:
        query: Search query string
        top_k: Number of results to return (default 5)

    Returns:
        List of document dicts with 'title', 'content', 'score' keys
    """
    # ... implementation ...
    return []

@tool
def run_sql_query(sql: str, database: str = "production") -> dict:
    """Execute a read-only SQL query against the analytics database.

    Args:
        sql: SQL SELECT statement to execute
        database: Target database name (default 'production')

    Returns:
        Dict with 'columns' list and 'rows' list of result data
    """
    # ... implementation ...
    return {"columns": [], "rows": []}

# When tool schemas are large, pass them as a cacheable block directly
# via the raw API request. In standard Strands usage the tool list is
# sent automatically — focus on keeping the system_prompt large enough
# (>1024 tokens) so the combined system+tools block crosses the threshold.

agent = Agent(
    model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0"),
    system_prompt="You are a data analyst...\n" + "detailed context " * 300,
    tools=[search_knowledge_base, run_sql_query]
)
```

For explicit control over tool caching in raw API calls, include the cache_point inside the tools array (Anthropic API format):

```python
# Raw Anthropic API format (for custom model providers or direct calls)
tools_with_cache = [
    {
        "name": "search_knowledge_base",
        "description": "Search internal knowledge base...",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string"},
                "top_k": {"type": "integer"}
            },
            "required": ["query"]
        }
    },
    # ... more tools ...
    {
        # Cache point after all tool definitions
        "cache_point": {"type": "default"}
    }
]
```

---

## Multi-Turn Caching: Document Context Across Turns

For document Q&A workflows, place the document in the first user turn with a cache_point. Subsequent turns extend the conversation without re-paying for the document.

```python
import asyncio
from strands import Agent
from strands.models import BedrockModel

async def document_qa_session(document_text: str, questions: list[str]) -> list[str]:
    agent = Agent(
        model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0"),
        system_prompt="You are a precise document analyst. Answer questions using only the provided document."
    )

    # Prime the conversation with the document, then cache it
    primer_messages = [
        {
            "role": "user",
            "content": [
                {"type": "text", "text": f"Document to analyze:\n\n{document_text}"},
                {"cache_point": {"type": "default"}}  # cache the document here
            ]
        },
        {
            "role": "assistant",
            "content": "I have read the document and am ready to answer your questions."
        }
    ]

    # Manually seed the conversation manager with the primed context
    agent.conversation_manager.messages.extend(primer_messages)

    answers = []
    for question in questions:
        # Each question call: document served from cache, only question billed at full rate
        result = await agent.invoke_async(question)
        answers.append(str(result))
        print(f"Q: {question}\nA: {str(result)[:100]}...\n")

    return answers

questions = [
    "What is the main thesis of the document?",
    "List all dates mentioned.",
    "Who are the key stakeholders identified?",
    "What are the three recommended next steps?",
]

asyncio.run(document_qa_session(open("report.txt").read(), questions))
```

**Cache hit pattern across turns:**

| Turn | Tokens billed at full rate | Tokens served from cache |
|---|---|---|
| Turn 1 (primer) | All (document + question) + cache write | 0 |
| Turn 2 | New question only (~20 tokens) | Document (~8,000 tokens) |
| Turn 3 | New question only (~20 tokens) | Document (~8,000 tokens) |
| Turn N (within TTL) | New question only (~20 tokens) | Document (~8,000 tokens) |

---

## Cache Warming Strategies

Cache warming means sending a request specifically to populate the cache before real user traffic arrives. This ensures the first real user request also gets a cache hit.

### Strategy 1 — Warm on application startup

```python
import asyncio
import logging
from strands import Agent
from strands.models import BedrockModel

logger = logging.getLogger(__name__)

async def warm_cache(agent: Agent) -> None:
    """Send a minimal warm-up request to populate the cache."""
    warm_prompt = "Hello."  # minimal real message; system prompt cache is populated regardless
    result = await agent.invoke_async(warm_prompt)
    usage = result.usage if hasattr(result, "usage") else {}
    created = usage.get("cache_creation_input_tokens", 0)
    read = usage.get("cache_read_input_tokens", 0)
    logger.info(f"Cache warm-up complete: created={created}, read={read}")

async def create_warmed_agent(system_prompt: str) -> Agent:
    agent = Agent(
        model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0"),
        system_prompt=system_prompt
    )
    await warm_cache(agent)
    return agent

# At application startup:
# agent = asyncio.run(create_warmed_agent(LARGE_SYSTEM_PROMPT))
```

### Strategy 2 — Scheduled re-warming to avoid TTL expiry

```python
import asyncio

async def cache_keeper(agent: Agent, interval_seconds: int = 240) -> None:
    """Re-warm cache every interval_seconds to prevent TTL expiry (default TTL is 300s)."""
    while True:
        await asyncio.sleep(interval_seconds)
        try:
            await agent.invoke_async("ping")
            logger.info("Cache re-warmed successfully")
        except Exception as e:
            logger.warning(f"Cache re-warm failed: {e}")

# Run as background task:
# asyncio.create_task(cache_keeper(agent, interval_seconds=240))
```

### Strategy 3 — Warm multiple variants in parallel

```python
async def warm_all_agents(agents: list[Agent]) -> None:
    """Warm multiple agents concurrently."""
    await asyncio.gather(*[warm_cache(a) for a in agents])
```

---

## Provider Differences: Bedrock vs Anthropic API

| Feature | Bedrock (`BedrockModel`) | Anthropic API (`AnthropicModel`) |
|---|---|---|
| Cache TTL | Varies by model/region; typically 5 min | 5 minutes |
| Minimum token threshold | 1024 tokens (same) | 1024 tokens |
| Cache write surcharge | ~25% of input rate (model-dependent) | 25% of base input rate |
| Cache read discount | ~10% of input rate (model-dependent) | 10% of base input rate |
| `cache_point` format | Same JSON structure | Same JSON structure |
| Supported models | Claude 3 Haiku+, 3.5 Sonnet/Haiku, 3.7 Sonnet | Same model family |
| Usage field names | Same (`cache_read_input_tokens`, etc.) | Same |
| Cross-region caching | No — cache is region-scoped | N/A (API is global) |

### Bedrock setup

```python
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",   # cache is scoped to this region
    max_tokens=4096
)
```

### Anthropic API setup

```python
from strands.models.anthropic import AnthropicModel
import os

model = AnthropicModel(
    model_id="claude-haiku-4-5-20251001",   # no "anthropic." prefix
    api_key=os.environ["ANTHROPIC_API_KEY"],
    max_tokens=4096
)
```

Both providers use the identical `cache_point` syntax in message content. The only user-facing difference is the model ID format and the source of credentials.

---

## Checking Cache Effectiveness

### Inspecting usage after a call

```python
result = agent("Your question here")
usage = result.usage  # dict or object depending on Strands version

cache_written = usage.get("cache_creation_input_tokens", 0) if isinstance(usage, dict) else getattr(usage, "cache_creation_input_tokens", 0)
cache_read    = usage.get("cache_read_input_tokens", 0)     if isinstance(usage, dict) else getattr(usage, "cache_read_input_tokens", 0)
input_tokens  = usage.get("input_tokens", 0)                if isinstance(usage, dict) else getattr(usage, "input_tokens", 0)

print(f"Tokens written to cache : {cache_written}")
print(f"Tokens served from cache: {cache_read}")
print(f"Tokens billed at full rate: {input_tokens}")

if cache_read > 0:
    total = cache_read + input_tokens
    hit_rate_pct = cache_read / total * 100
    print(f"Cache hit ratio: {hit_rate_pct:.1f}%")
else:
    print("No cache hit — first call or cache expired")
```

### Before/after comparison helper

```python
import time
from dataclasses import dataclass
from strands import Agent

@dataclass
class CacheStats:
    call_index: int
    elapsed_ms: float
    input_tokens: int
    cache_written: int
    cache_read: int

    @property
    def cache_hit(self) -> bool:
        return self.cache_read > 0

    @property
    def effective_cost_tokens(self) -> float:
        """Weighted token count: cache reads at 10% cost, writes at 125%, normal at 100%."""
        return self.input_tokens + self.cache_written * 1.25 + self.cache_read * 0.10

def benchmark_caching(agent: Agent, prompts: list[str]) -> list[CacheStats]:
    stats = []
    for i, prompt in enumerate(prompts):
        t0 = time.monotonic()
        result = agent(prompt)
        elapsed = (time.monotonic() - t0) * 1000
        usage = result.usage
        if isinstance(usage, dict):
            get = lambda k: usage.get(k, 0)
        else:
            get = lambda k: getattr(usage, k, 0)
        s = CacheStats(
            call_index=i,
            elapsed_ms=elapsed,
            input_tokens=get("input_tokens"),
            cache_written=get("cache_creation_input_tokens"),
            cache_read=get("cache_read_input_tokens"),
        )
        stats.append(s)
        status = "HIT" if s.cache_hit else ("WRITE" if s.cache_written > 0 else "MISS")
        print(f"[{i}] {status:5s} | {elapsed:6.0f}ms | in={s.input_tokens:5d} | written={s.cache_written:5d} | read={s.cache_read:5d} | eff={s.effective_cost_tokens:6.0f}")
    return stats

# Usage:
# stats = benchmark_caching(agent, ["Question 1", "Question 2", "Question 3"])
```

---

## Structuring Prompts to Maximize Cache Hits

The cache key is the **exact token prefix** before the cache_point. Every byte difference breaks the cache. These rules maximize hit rates:

### Rule 1 — Static content first, dynamic content last

```python
# BAD: dynamic session_id at the top breaks cache on every call
system_prompt_bad = f"""
Session: {session_id}
You are a coding assistant with the following knowledge base:
[...10,000 tokens of static documentation...]
"""

# GOOD: all dynamic content goes after the cache boundary
system_prompt_good = """
You are a coding assistant with the following knowledge base:
[...10,000 tokens of static documentation...]
"""
# Session context goes in the user message, not the system prompt
```

### Rule 2 — Keep the cached prefix identical across agents sharing the same context

```python
# GOOD: shared base prompt, variations appended after a delimiter
BASE_SYSTEM_PROMPT = """
[10,000 tokens of shared company knowledge base]
""".strip()

def make_agent(role_instructions: str) -> Agent:
    # role_instructions are SHORT and go AFTER the large shared block
    # The large shared block is the same across all agents → shared cache slot
    full_prompt = BASE_SYSTEM_PROMPT + "\n\n" + role_instructions
    return Agent(
        model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0"),
        system_prompt=full_prompt
    )

support_agent = make_agent("You handle billing inquiries. Be concise and empathetic.")
sales_agent   = make_agent("You handle product questions. Be enthusiastic and informative.")
```

### Rule 3 — Do not insert timestamps or UUIDs into the cached prefix

```python
# BAD: timestamp changes every second → permanent cache miss
system_prompt_bad = f"Current time: {datetime.utcnow().isoformat()}\n\nYou are a helpful assistant..."

# GOOD: pass dynamic context only in the user turn
system_prompt_good = "You are a helpful assistant..."
user_message = f"[Context: current time is {datetime.utcnow().isoformat()}]\n\nUser query: {user_query}"
```

### Rule 4 — Batch related questions within the TTL window

If you need to ask 20 questions about the same document, send them in rapid succession (within 5 minutes) so all 20 benefit from the same cache population.

---

## Cache Invalidation Scenarios

Understanding when the cache is NOT hit prevents surprises:

| Scenario | Why cache misses | Fix |
|---|---|---|
| First call ever | Cache has not been populated yet | Expected; pay cache-write rate once |
| >5 minutes between calls | TTL expired; cache evicted | Use cache keeper background task |
| System prompt changed (even one character) | Token prefix hash differs | Keep cached prefix strictly static |
| Timestamp/UUID in system prompt | Prefix changes every call | Move dynamic data to user message |
| Different model ID | Cache is per-model | Ensure `model_id` is identical across calls |
| Different AWS region | Bedrock cache is region-scoped | Pin to one region for cached workloads |
| Content block order changed | Token sequence changes | Keep block order stable |
| New tool added before cache_point | Changes token sequence | Append new tools after cache_point or rebuild cache |
| Cache_point below 1024 tokens | Minimum not met; cache_point ignored silently | Expand context or consolidate blocks |
| Image content changed | Binary hash differs | Use stable image references if possible |

---

## Production Pattern: Cache-Aware Agent Factory

A production service typically needs to:
1. Create agents once with a warmed cache
2. Reuse them across requests (avoiding per-request Agent construction overhead)
3. Monitor cache hit rates in metrics
4. Re-warm periodically to prevent TTL expiry

```python
import asyncio
import logging
import time
from dataclasses import dataclass, field
from typing import Optional
from strands import Agent
from strands.models import BedrockModel

logger = logging.getLogger(__name__)


@dataclass
class CacheMetrics:
    total_calls: int = 0
    cache_hits: int = 0
    cache_misses: int = 0
    total_tokens_saved: int = 0

    @property
    def hit_rate(self) -> float:
        if self.total_calls == 0:
            return 0.0
        return self.cache_hits / self.total_calls

    def record(self, cache_read: int, cache_written: int) -> None:
        self.total_calls += 1
        if cache_read > 0:
            self.cache_hits += 1
            self.total_tokens_saved += cache_read
        elif cache_written > 0:
            self.cache_misses += 1  # cache write = first call or expiry
        else:
            self.cache_misses += 1


class CacheAwareAgentFactory:
    """Manages a pool of pre-warmed agents for production use."""

    def __init__(
        self,
        system_prompt: str,
        model_id: str = "anthropic.claude-haiku-4-5-20251001-v1:0",
        region: str = "us-east-1",
        rewarm_interval_seconds: int = 240,
    ):
        self.system_prompt = system_prompt
        self.model_id = model_id
        self.region = region
        self.rewarm_interval = rewarm_interval_seconds
        self._agent: Optional[Agent] = None
        self._metrics = CacheMetrics()
        self._rewarm_task: Optional[asyncio.Task] = None

    async def initialize(self) -> None:
        """Create agent and warm the cache. Call once at startup."""
        self._agent = Agent(
            model=BedrockModel(model_id=self.model_id, region_name=self.region),
            system_prompt=self.system_prompt,
        )
        await self._warm()
        self._rewarm_task = asyncio.create_task(self._rewarm_loop())
        logger.info("CacheAwareAgentFactory initialized and cache warmed")

    async def _warm(self) -> None:
        result = await self._agent.invoke_async("Hello.")
        usage = result.usage
        get = (lambda k: usage.get(k, 0)) if isinstance(usage, dict) else (lambda k: getattr(usage, k, 0))
        logger.info(
            f"Cache warm: written={get('cache_creation_input_tokens')}, "
            f"read={get('cache_read_input_tokens')}"
        )

    async def _rewarm_loop(self) -> None:
        while True:
            await asyncio.sleep(self.rewarm_interval)
            try:
                await self._warm()
            except Exception as e:
                logger.warning(f"Cache re-warm failed: {e}")

    async def invoke(self, prompt: str) -> str:
        if self._agent is None:
            raise RuntimeError("Factory not initialized. Call initialize() first.")
        result = await self._agent.invoke_async(prompt)
        usage = result.usage
        get = (lambda k: usage.get(k, 0)) if isinstance(usage, dict) else (lambda k: getattr(usage, k, 0))
        self._metrics.record(
            cache_read=get("cache_read_input_tokens"),
            cache_written=get("cache_creation_input_tokens"),
        )
        return str(result)

    def get_metrics(self) -> dict:
        return {
            "total_calls": self._metrics.total_calls,
            "cache_hit_rate": f"{self._metrics.hit_rate:.1%}",
            "cache_hits": self._metrics.cache_hits,
            "cache_misses": self._metrics.cache_misses,
            "total_tokens_saved": self._metrics.total_tokens_saved,
        }

    async def shutdown(self) -> None:
        if self._rewarm_task:
            self._rewarm_task.cancel()


# --- Usage in a FastAPI app ---
# factory = CacheAwareAgentFactory(system_prompt=LARGE_PROMPT)
#
# @app.on_event("startup")
# async def startup():
#     await factory.initialize()
#
# @app.post("/ask")
# async def ask(question: str):
#     answer = await factory.invoke(question)
#     return {"answer": answer, "cache_metrics": factory.get_metrics()}
```

---

## Anti-Patterns That Break Caching

### Anti-pattern 1 — Creating a new Agent per request

```python
# BAD: new Agent object on every request means a cold cache every time
@app.post("/ask")
async def ask(question: str):
    agent = Agent(model=BedrockModel(...), system_prompt=LARGE_PROMPT)  # cold every call
    return str(await agent.invoke_async(question))

# GOOD: create once, reuse; clear conversation state between users
agent = Agent(model=BedrockModel(...), system_prompt=LARGE_PROMPT)

@app.post("/ask")
async def ask(question: str):
    agent.conversation_manager.clear()
    return str(await agent.invoke_async(question))
```

### Anti-pattern 2 — Mutating system prompt per user

```python
# BAD: per-user data injected into system prompt → unique prefix per user → no cache sharing
def bad_agent_for_user(user_name: str, user_id: str) -> Agent:
    return Agent(
        system_prompt=f"User: {user_name} (id={user_id})\n\n[10,000 tokens of shared docs]"
    )

# GOOD: shared static system prompt → single warm cache for all users
SHARED_SYSTEM_PROMPT = "[10,000 tokens of shared docs]"

def good_agent_for_user(user_name: str) -> str:
    # User context goes in the user turn, not the system prompt
    return f"[Answering for user: {user_name}] "
```

### Anti-pattern 3 — cache_point not at the end of the content array

```python
# BAD: cache_point must be the last element; anything after it is uncached
bad_content = [
    {"cache_point": {"type": "default"}},   # wrong position
    {"type": "text", "text": large_document}
]

# GOOD: cache_point is last
good_content = [
    {"type": "text", "text": large_document},
    {"cache_point": {"type": "default"}}    # correct — caches everything above
]
```

### Anti-pattern 4 — Placing cache_point inside a short block

```python
# BAD: if document is only 500 tokens, cache_point does nothing (<1024 token min)
short_content = [
    {"type": "text", "text": "Short intro paragraph."},  # ~50 tokens
    {"cache_point": {"type": "default"}}  # silently ignored
]

# GOOD: ensure the block before cache_point exceeds 1024 tokens
long_content = [
    {"type": "text", "text": large_reference_document},  # >1024 tokens
    {"cache_point": {"type": "default"}}
]
```

### Anti-pattern 5 — Forgetting that Bedrock caches are region-scoped

```python
# BAD: load-balancing across regions defeats cache
regions = ["us-east-1", "us-west-2", "eu-west-1"]
model = BedrockModel(region_name=random.choice(regions))  # different cache per region

# GOOD: pin to one region for cached workloads
model = BedrockModel(region_name="us-east-1")
```

---

## Troubleshooting

| Symptom | Root Cause | Diagnosis | Fix |
|---|---|---|---|
| `cache_read_input_tokens` always 0 | Cache never hit | Check token count before cache_point | Ensure >1024 tokens before cache_point |
| `cache_read_input_tokens` 0 after first call | TTL expired between calls | Calls more than 5 min apart | Add cache keeper re-warm task |
| `cache_creation_input_tokens` 0 AND `cache_read_input_tokens` 0 | cache_point silently ignored | Content too short, or wrong placement | Count tokens; move cache_point to end of content array |
| Cache hits on Anthropic API but not Bedrock | Model version unsupported on Bedrock | Not all Bedrock model ARNs support caching | Check Bedrock docs for supported model versions in your region |
| Cache stops working after code change | System prompt or tool definition changed | One-char diff breaks the cache key | Diff the system_prompt string before and after change |
| Unexpected cost increase after enabling caching | Cache write surcharge on every call | Check if TTL is expiring before next call | Inspect `cache_creation_input_tokens`; reduce call interval or add re-warm |
| `cache_point` key error in response | Wrong dict key spelling | API validation error | Use exact key: `"cache_point"` (underscore, not hyphen) |
| High latency on cache miss | Normal — first call computes full prefix | Compare cold vs warm timings | Accept first-call penalty; subsequent calls will be faster |
| Multi-region deployment has inconsistent cache hit rates | Each region has its own cache | Some requests route to uncached region | Route cached workloads to a single fixed region |
