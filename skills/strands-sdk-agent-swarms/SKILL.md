---
name: strands-sdk-agent-swarms
description: "Use when running multiple Strands agents in parallel on the same or similar tasks, implementing fan-out/fan-in patterns, processing batches concurrently, or building swarm intelligence workflows with result aggregation. Triggers on: swarm, parallel agents, fan-out, asyncio.gather, batch processing, concurrent agents, fan-in synthesis."
---

# Agent Swarms — Strands SDK

## Overview

Swarms run many agents concurrently on the same or similar tasks, then aggregate results. Use Python's `asyncio.gather` to fan out to multiple agents and collect all responses. Swarms can be homogeneous (identical agents) or heterogeneous (specialized agents), fixed-size or dynamically sized, and can aggregate results through voting, averaging, or LLM-based synthesis.

## Quick Reference

| Pattern | Tool | Use Case |
|---|---|---|
| Fan-out | `asyncio.gather` | Same prompt, multiple agents |
| Fan-out/fan-in | `gather` + aggregator agent | Parallel work, then synthesis |
| Batch processing | `gather` over list | Different inputs, same agent type |
| Rate-limited swarm | `asyncio.Semaphore` | Cap concurrency, avoid throttling |
| Pipeline | Sequential `gather` chains | Staged processing, each stage fans out |
| Mesh | All-to-all communication | Agents refine each other's output |
| Hierarchical | Supervisor + sub-swarms | Complex decomposition, team-of-teams |
| Dynamic sizing | Input-driven agent count | Scale workers to problem complexity |

---

## Swarm Topologies

### 1. Star Topology (Fan-Out / Fan-In)

One coordinator dispatches to N workers; all results return to the coordinator for aggregation. This is the most common pattern.

```
           ┌─── Worker A ───┐
Coordinator─┼─── Worker B ───┼─► Aggregator
           └─── Worker C ───┘
```

```python
import asyncio
from strands import Agent
from strands.models import BedrockModel

def make_model():
    return BedrockModel(
        model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0",
        region_name="us-east-1",
    )

async def star_swarm(prompt: str) -> str:
    workers = [
        Agent(model=make_model(), system_prompt="You are optimistic. Identify opportunities and positives."),
        Agent(model=make_model(), system_prompt="You are a devil's advocate. Identify risks and failure modes."),
        Agent(model=make_model(), system_prompt="You are pragmatic. Identify trade-offs and implementation concerns."),
    ]

    results = await asyncio.gather(*[w.invoke_async(prompt) for w in workers])

    aggregator = Agent(
        model=make_model(),
        system_prompt="Synthesize multiple expert perspectives into a balanced, actionable recommendation."
    )
    combined = "\n\n".join(f"Perspective {i+1}:\n{r}" for i, r in enumerate(results))
    final = await aggregator.invoke_async(
        f"Synthesize these perspectives on the question: '{prompt}'\n\n{combined}"
    )
    return str(final)

result = asyncio.run(star_swarm("Should we adopt microservices architecture?"))
```

---

### 2. Pipeline Topology

Output of one stage feeds the next stage. Each stage can itself be a swarm. Use this for multi-step transformations where intermediate results are refined.

```
Stage 1 (research)  ──►  Stage 2 (analysis)  ──►  Stage 3 (report)
[A1, A2, A3]             [B1, B2]                  [C1]
```

```python
async def pipeline_swarm(topic: str) -> str:
    # Stage 1: Research — 3 agents gather different facets
    research_agents = [
        Agent(model=make_model(), system_prompt="Research technical aspects only."),
        Agent(model=make_model(), system_prompt="Research business and market aspects only."),
        Agent(model=make_model(), system_prompt="Research regulatory and compliance aspects only."),
    ]
    research_results = await asyncio.gather(
        *[a.invoke_async(f"Research this topic: {topic}") for a in research_agents]
    )
    research_summary = "\n\n".join(str(r) for r in research_results)

    # Stage 2: Analysis — 2 agents analyze different dimensions of the research
    analysis_agents = [
        Agent(model=make_model(), system_prompt="Perform SWOT analysis based on provided research."),
        Agent(model=make_model(), system_prompt="Perform cost-benefit analysis based on provided research."),
    ]
    analysis_results = await asyncio.gather(
        *[a.invoke_async(f"Analyze this research:\n\n{research_summary}") for a in analysis_agents]
    )
    analysis_summary = "\n\n".join(str(r) for r in analysis_results)

    # Stage 3: Single report writer synthesizes everything
    reporter = Agent(
        model=make_model(),
        system_prompt="Write a concise executive report with clear recommendations."
    )
    report = await reporter.invoke_async(
        f"Write an executive report on '{topic}' using this analysis:\n\n{analysis_summary}"
    )
    return str(report)
```

---

### 3. Mesh Topology

Each agent sees the outputs of all others and iteratively refines. Use for consensus-building, peer review, or adversarial refinement. Run multiple rounds until convergence or a fixed iteration count.

```
A ◄──► B
▲       ▲
└──► C ─┘
```

```python
async def mesh_swarm(prompt: str, rounds: int = 2) -> list[str]:
    agents = [
        Agent(model=make_model(), system_prompt="Expert A: architecture focus."),
        Agent(model=make_model(), system_prompt="Expert B: security focus."),
        Agent(model=make_model(), system_prompt="Expert C: performance focus."),
    ]

    # Round 0: initial independent responses
    responses = list(await asyncio.gather(*[a.invoke_async(prompt) for a in agents]))
    responses = [str(r) for r in responses]

    # Subsequent rounds: each agent refines given all peers' previous outputs
    for round_num in range(1, rounds + 1):
        peer_context = "\n\n".join(
            f"Peer {i+1} said:\n{r}" for i, r in enumerate(responses)
        )
        refined = await asyncio.gather(*[
            agents[i].invoke_async(
                f"Original question: {prompt}\n\n"
                f"Your peers responded:\n{peer_context}\n\n"
                f"Refine or defend your position (round {round_num})."
            )
            for i in range(len(agents))
        ])
        responses = [str(r) for r in refined]

    return responses
```

---

### 4. Hierarchical Topology

A top-level supervisor decomposes the task into sub-tasks, each handled by a sub-swarm. Results bubble up. Use for very complex tasks requiring decomposition into independent work streams.

```
           Supervisor
          /     |     \
       Team A  Team B  Team C
       /  \    /  \    /  \
      A1  A2  B1  B2  C1  C2
```

```python
async def team_swarm(task: str, specialists: list[str]) -> str:
    """One sub-team: multiple specialists tackle the same sub-task."""
    agents = [
        Agent(model=make_model(), system_prompt=f"You are a {role}. Focus on your domain.")
        for role in specialists
    ]
    results = await asyncio.gather(*[a.invoke_async(task) for a in agents])
    # Mini-aggregator for this team
    agg = Agent(model=make_model(), system_prompt="Summarize the team's findings concisely.")
    summary = await agg.invoke_async(
        "\n\n".join(f"{specialists[i]}: {results[i]}" for i in range(len(specialists)))
    )
    return str(summary)

async def hierarchical_swarm(project: str) -> str:
    supervisor = Agent(
        model=make_model(),
        system_prompt="Decompose a project into 3 independent work streams as a JSON list of dicts with keys 'task' and 'team'."
    )
    decomposition_raw = await supervisor.invoke_async(
        f"Decompose this project into independent streams: {project}\n"
        "Return JSON: [{\"task\": \"...\", \"team\": [\"role1\", \"role2\"]}]"
    )

    import json, re
    match = re.search(r'\[.*\]', str(decomposition_raw), re.DOTALL)
    streams = json.loads(match.group()) if match else []

    # Run each sub-team in parallel
    team_results = await asyncio.gather(*[
        team_swarm(stream["task"], stream["team"])
        for stream in streams
    ])

    # Top-level integration
    integrator = Agent(
        model=make_model(),
        system_prompt="Integrate sub-team results into a unified project plan."
    )
    final = await integrator.invoke_async(
        f"Project: {project}\n\nSub-team results:\n\n" +
        "\n\n".join(f"Stream {i+1}: {r}" for i, r in enumerate(team_results))
    )
    return str(final)
```

---

## Dynamic Swarm Sizing

Scale the number of agents based on the complexity or volume of the input. Avoid over-provisioning for simple inputs; scale up for complex ones.

```python
async def dynamic_swarm(items: list[str], complexity_hint: str = "medium") -> list[str]:
    """
    Adjusts worker count based on complexity and input size.
    complexity_hint: "low" | "medium" | "high"
    """
    complexity_multiplier = {"low": 1, "medium": 2, "high": 3}
    base_workers = complexity_multiplier.get(complexity_hint, 2)
    # Never spin up more workers than items; cap at 10 to control cost
    num_workers = min(len(items), base_workers * 2, 10)

    semaphore = asyncio.Semaphore(num_workers)

    async def run_one(item: str) -> str:
        async with semaphore:
            agent = Agent(
                model=make_model(),
                system_prompt="Process the given item thoroughly."
            )
            return str(await agent.invoke_async(item))

    return list(await asyncio.gather(*[run_one(item) for item in items]))


async def size_swarm_by_content(document: str) -> str:
    """Dynamically size the swarm based on document word count."""
    word_count = len(document.split())
    # ~1 agent per 500 words, between 2 and 8 agents
    num_agents = max(2, min(8, word_count // 500))

    chunk_size = len(document) // num_agents
    chunks = [document[i * chunk_size:(i + 1) * chunk_size] for i in range(num_agents)]
    # Give the last agent any leftover text
    chunks[-1] += document[num_agents * chunk_size:]

    agents = [
        Agent(model=make_model(), system_prompt="Summarize this text section concisely.")
        for _ in range(num_agents)
    ]
    summaries = await asyncio.gather(*[
        agents[i].invoke_async(f"Summarize:\n\n{chunks[i]}") for i in range(num_agents)
    ])

    merger = Agent(model=make_model(), system_prompt="Merge section summaries into a coherent whole.")
    merged = await merger.invoke_async("\n\n".join(str(s) for s in summaries))
    return str(merged)
```

---

## Heterogeneous Swarms

Mix agents with different models, tools, system prompts, or memory configurations in the same swarm. Each agent type brings a distinct capability.

```python
from strands.tools import calculator, web_search  # illustrative tool imports

async def heterogeneous_swarm(query: str) -> dict:
    """
    Three agent types: fast/cheap model for drafting,
    powerful model for deep reasoning, tool-equipped agent for lookup.
    """
    # Fast drafter — cheap model, high throughput
    drafter = Agent(
        model=BedrockModel(
            model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0",
            region_name="us-east-1",
        ),
        system_prompt="Draft a quick initial answer. Prioritize speed over depth."
    )

    # Deep reasoner — more capable model, slower
    reasoner = Agent(
        model=BedrockModel(
            model_id="us.anthropic.claude-sonnet-4-5-20251001-v1:0",
            region_name="us-east-1",
        ),
        system_prompt="Provide deep, nuanced reasoning. Take your time."
    )

    # Tool-equipped fact-checker
    fact_checker = Agent(
        model=make_model(),
        system_prompt="Verify factual claims. Flag anything uncertain.",
        tools=[web_search],  # has access to external search
    )

    draft_task = drafter.invoke_async(query)
    reason_task = reasoner.invoke_async(query)
    fact_task = fact_checker.invoke_async(f"Check facts in this query: {query}")

    draft, reasoning, facts = await asyncio.gather(draft_task, reason_task, fact_task)

    return {
        "draft": str(draft),
        "deep_reasoning": str(reasoning),
        "fact_check": str(facts),
    }
```

---

## Result Aggregation Strategies

### Voting (Majority / Plurality)

For classification or discrete-choice tasks where agents vote on an answer.

```python
from collections import Counter

async def voting_swarm(question: str, num_voters: int = 5) -> str:
    """Run N agents and return the most common answer."""
    agents = [
        Agent(
            model=make_model(),
            system_prompt=(
                "Answer with ONLY a single word or short phrase — no explanation. "
                "Choose the single best option."
            )
        )
        for _ in range(num_voters)
    ]
    votes_raw = await asyncio.gather(*[a.invoke_async(question) for a in agents])
    votes = [str(v).strip().lower() for v in votes_raw]

    tally = Counter(votes)
    winner, count = tally.most_common(1)[0]
    confidence = count / num_voters
    return f"{winner} (confidence: {confidence:.0%}, votes: {tally})"
```

### Numeric Averaging

For tasks that return numeric scores (e.g., quality ratings, sentiment scores).

```python
import re as _re

async def averaging_swarm(text: str, num_raters: int = 5) -> dict:
    """Have N agents rate text quality 1-10; return mean and std dev."""
    agents = [
        Agent(
            model=make_model(),
            system_prompt="Rate the quality of the text on a scale of 1-10. Respond with ONLY the integer."
        )
        for _ in range(num_raters)
    ]
    raw_results = await asyncio.gather(*[
        a.invoke_async(f"Rate this text:\n\n{text}") for a in agents
    ])

    scores = []
    for r in raw_results:
        match = _re.search(r'\b([1-9]|10)\b', str(r))
        if match:
            scores.append(int(match.group()))

    if not scores:
        return {"mean": None, "std": None, "raw": [str(r) for r in raw_results]}

    mean = sum(scores) / len(scores)
    variance = sum((s - mean) ** 2 for s in scores) / len(scores)
    std = variance ** 0.5
    return {"mean": round(mean, 2), "std": round(std, 2), "scores": scores}
```

### LLM Synthesis (Fan-In)

The richest aggregation: a dedicated agent reads all worker outputs and synthesizes a coherent response.

```python
async def synthesis_swarm(prompt: str, perspectives: list[str] | None = None) -> str:
    """Fan out to perspective agents, then fan in with a synthesis agent."""
    if perspectives is None:
        perspectives = ["optimistic", "critical", "pragmatic", "creative", "risk-focused"]

    workers = [
        Agent(
            model=make_model(),
            system_prompt=f"You are a {p} thinker. Analyze from your perspective only."
        )
        for p in perspectives
    ]
    results = await asyncio.gather(*[w.invoke_async(prompt) for w in workers])

    synthesizer = Agent(
        model=make_model(),
        system_prompt=(
            "You are a senior analyst. Synthesize multiple expert perspectives into a "
            "concise, balanced recommendation. Highlight agreements, tensions, and your "
            "final recommendation."
        )
    )
    labeled = "\n\n".join(
        f"[{perspectives[i].upper()} VIEW]\n{results[i]}" for i in range(len(perspectives))
    )
    synthesis = await synthesizer.invoke_async(
        f"Question: {prompt}\n\nExpert perspectives:\n\n{labeled}\n\n"
        "Provide your synthesis and recommendation."
    )
    return str(synthesis)
```

---

## Partial Failure Handling

When some swarm workers fail, continue with the rest rather than cancelling the entire swarm.

```python
import logging

logger = logging.getLogger(__name__)

async def resilient_swarm(
    prompt: str,
    agents: list[Agent],
    min_successes: int = 2,
) -> list[str]:
    """
    Run all agents; collect successes. Raise only if fewer than min_successes succeed.
    """
    async def safe_invoke(agent: Agent, idx: int) -> tuple[int, str | None]:
        try:
            result = await agent.invoke_async(prompt)
            return idx, str(result)
        except Exception as exc:
            logger.warning("Swarm worker %d failed: %s", idx, exc)
            return idx, None

    raw = await asyncio.gather(*[safe_invoke(a, i) for i, a in enumerate(agents)])

    successes = [(idx, res) for idx, res in raw if res is not None]
    failures  = [(idx, ) for idx, res in raw if res is None]

    if failures:
        logger.warning("%d/%d swarm workers failed: indices %s",
                       len(failures), len(agents), [f[0] for f in failures])

    if len(successes) < min_successes:
        raise RuntimeError(
            f"Too many swarm failures: only {len(successes)}/{len(agents)} succeeded "
            f"(minimum required: {min_successes})"
        )

    # Return results in original index order
    return [res for _, res in sorted(successes)]


async def resilient_swarm_with_retry(
    items: list[str],
    max_retries: int = 2,
    max_concurrent: int = 5,
) -> list[str | None]:
    """Process a batch; retry failed items up to max_retries times."""
    semaphore = asyncio.Semaphore(max_concurrent)
    results: list[str | None] = [None] * len(items)

    async def attempt(idx: int, item: str) -> None:
        async with semaphore:
            for attempt_num in range(1, max_retries + 1):
                try:
                    agent = Agent(model=make_model(), system_prompt="Process the item.")
                    results[idx] = str(await agent.invoke_async(item))
                    return
                except Exception as exc:
                    logger.warning(
                        "Item %d attempt %d/%d failed: %s", idx, attempt_num, max_retries, exc
                    )
                    if attempt_num < max_retries:
                        await asyncio.sleep(2 ** attempt_num)  # exponential back-off
            logger.error("Item %d exhausted retries.", idx)

    await asyncio.gather(*[attempt(i, item) for i, item in enumerate(items)])
    return results
```

---

## Progress Reporting Across a Swarm

Use `asyncio.Queue` or a shared counter to emit progress updates while workers are running.

```python
import asyncio
from dataclasses import dataclass, field
from typing import Callable

@dataclass
class SwarmProgress:
    total: int
    completed: int = 0
    failed: int = 0
    _lock: asyncio.Lock = field(default_factory=asyncio.Lock, repr=False)

    async def record_success(self) -> None:
        async with self._lock:
            self.completed += 1

    async def record_failure(self) -> None:
        async with self._lock:
            self.failed += 1

    @property
    def pct(self) -> float:
        return (self.completed + self.failed) / self.total * 100


async def swarm_with_progress(
    items: list[str],
    on_progress: Callable[[SwarmProgress], None] | None = None,
    max_concurrent: int = 5,
) -> list[str | None]:
    semaphore = asyncio.Semaphore(max_concurrent)
    progress = SwarmProgress(total=len(items))
    results: list[str | None] = [None] * len(items)

    async def run_one(idx: int, item: str) -> None:
        async with semaphore:
            try:
                agent = Agent(model=make_model(), system_prompt="Process the item.")
                results[idx] = str(await agent.invoke_async(item))
                await progress.record_success()
            except Exception as exc:
                logger.warning("Worker %d failed: %s", idx, exc)
                await progress.record_failure()
            if on_progress:
                on_progress(progress)

    await asyncio.gather(*[run_one(i, item) for i, item in enumerate(items)])
    return results


# Usage
def print_progress(p: SwarmProgress) -> None:
    print(f"Progress: {p.completed}/{p.total} done, {p.failed} failed ({p.pct:.0f}%)")

results = asyncio.run(
    swarm_with_progress(["item 1", "item 2", "item 3"], on_progress=print_progress)
)
```

---

## Memory-Efficient Swarms

For large batches, process and discard results incrementally rather than accumulating everything in memory. Use `asyncio.as_completed` or chunked batches.

```python
async def memory_efficient_swarm(
    items: list[str],
    output_path: str,
    batch_size: int = 10,
    max_concurrent: int = 5,
) -> int:
    """
    Process items in fixed batches. Write each batch's results to disk immediately.
    Never holds more than batch_size results in memory at once.
    Returns total number of successfully processed items.
    """
    semaphore = asyncio.Semaphore(max_concurrent)
    total_processed = 0

    async def run_one(item: str) -> str | None:
        async with semaphore:
            try:
                agent = Agent(model=make_model(), system_prompt="Summarize this text.")
                return str(await agent.invoke_async(item))
            except Exception as exc:
                logger.warning("Failed item: %s", exc)
                return None

    for batch_start in range(0, len(items), batch_size):
        batch = items[batch_start : batch_start + batch_size]
        batch_results = await asyncio.gather(*[run_one(item) for item in batch])

        # Write immediately, then let GC reclaim
        with open(output_path, "a", encoding="utf-8") as fh:
            for result in batch_results:
                if result is not None:
                    fh.write(result + "\n")
                    total_processed += 1

        # Explicitly release references before next batch
        del batch_results

    return total_processed


async def streaming_swarm(items: list[str]) -> None:
    """
    Use as_completed to process results as they arrive — no waiting for stragglers.
    Suitable for streaming output to a consumer (e.g., a WebSocket).
    """
    async def run_one(item: str, idx: int) -> tuple[int, str]:
        agent = Agent(model=make_model(), system_prompt="Process concisely.")
        result = await agent.invoke_async(item)
        return idx, str(result)

    tasks = [asyncio.create_task(run_one(item, i)) for i, item in enumerate(items)]

    for coro in asyncio.as_completed(tasks):
        idx, result = await coro
        # Emit immediately — do not accumulate
        print(f"[{idx}] {result[:120]}...")
```

---

## Swarms with Shared Tools

Give all agents in a swarm access to the same tool set. Tools are generally stateless functions so sharing is safe; beware of tools with shared mutable state.

```python
from strands import tool

@tool
def search_knowledge_base(query: str) -> str:
    """Search the internal knowledge base."""
    # Replace with your actual KB lookup
    return f"KB results for '{query}': [result placeholder]"

@tool
def fetch_url(url: str) -> str:
    """Fetch and return the text content of a URL."""
    import urllib.request
    try:
        with urllib.request.urlopen(url, timeout=10) as resp:
            return resp.read().decode("utf-8", errors="replace")[:4000]
    except Exception as exc:
        return f"Error fetching {url}: {exc}"

async def tool_equipped_swarm(topics: list[str]) -> list[str]:
    """Each worker has access to shared read-only tools."""
    SHARED_TOOLS = [search_knowledge_base, fetch_url]

    async def run_one(topic: str) -> str:
        agent = Agent(
            model=make_model(),
            system_prompt="Research the topic using available tools, then summarize.",
            tools=SHARED_TOOLS,
        )
        return str(await agent.invoke_async(f"Research: {topic}"))

    return list(await asyncio.gather(*[run_one(topic) for topic in topics]))
```

**Caution:** Tools with mutable shared state (e.g., a shared in-memory counter, a file being written) require explicit locking. Prefer pure/read-only tools in swarms.

---

## Real-World Examples

### Example 1: Document Processing Swarm

Process a large corpus of documents in parallel — extract key information from each, then produce an overall summary.

```python
async def document_processing_swarm(documents: list[dict]) -> dict:
    """
    documents: list of {"id": str, "text": str, "type": str}
    Returns per-document extractions and a corpus-level summary.
    """
    semaphore = asyncio.Semaphore(6)

    async def extract_one(doc: dict) -> dict:
        async with semaphore:
            agent = Agent(
                model=make_model(),
                system_prompt=(
                    "Extract: (1) main topic, (2) key entities, (3) sentiment, "
                    "(4) a 2-sentence summary. Return as JSON."
                )
            )
            raw = await agent.invoke_async(
                f"Document type: {doc['type']}\n\nContent:\n{doc['text'][:3000]}"
            )
            import json, re
            match = re.search(r'\{.*\}', str(raw), re.DOTALL)
            extraction = json.loads(match.group()) if match else {"raw": str(raw)}
            return {"id": doc["id"], **extraction}

    extractions = list(await asyncio.gather(*[extract_one(doc) for doc in documents]))

    # Corpus-level summary
    corpus_agent = Agent(
        model=make_model(),
        system_prompt="Identify cross-document themes, patterns, and anomalies."
    )
    summaries_text = "\n".join(
        f"Doc {e['id']}: {e.get('summary', e.get('raw', ''))[:200]}"
        for e in extractions
    )
    corpus_summary = await corpus_agent.invoke_async(
        f"Analyze {len(documents)} documents:\n\n{summaries_text}"
    )
    return {"documents": extractions, "corpus_summary": str(corpus_summary)}
```

---

### Example 2: Multi-Perspective Analysis Swarm

Generate a rich, balanced analysis of a topic by assigning distinct analytical lenses to specialist agents.

```python
ANALYST_ROLES = {
    "technical":    "Focus on technical feasibility, architecture, and implementation.",
    "financial":    "Focus on costs, ROI, cash flow, and financial risk.",
    "legal":        "Focus on regulatory compliance, liability, and legal risk.",
    "user":         "Focus on user experience, adoption, and satisfaction.",
    "competitive":  "Focus on market positioning and competitive dynamics.",
    "ethical":      "Focus on ethical implications and societal impact.",
}

async def multi_perspective_analysis(topic: str, roles: list[str] | None = None) -> dict:
    selected_roles = roles or list(ANALYST_ROLES.keys())
    agents = {
        role: Agent(
            model=make_model(),
            system_prompt=f"You are a {role} analyst. {ANALYST_ROLES[role]} Be concise and specific."
        )
        for role in selected_roles
    }

    tasks = {role: agent.invoke_async(topic) for role, agent in agents.items()}
    raw_results = dict(zip(tasks.keys(), await asyncio.gather(*tasks.values())))
    perspectives = {role: str(result) for role, result in raw_results.items()}

    # Conflict detector
    conflict_agent = Agent(
        model=make_model(),
        system_prompt="Identify key tensions and conflicts between different analytical perspectives."
    )
    perspectives_text = "\n\n".join(f"[{r.upper()}]\n{v}" for r, v in perspectives.items())
    conflicts = await conflict_agent.invoke_async(perspectives_text)

    # Final recommendation
    rec_agent = Agent(
        model=make_model(),
        system_prompt="Given multi-perspective analysis and identified conflicts, provide a balanced recommendation."
    )
    recommendation = await rec_agent.invoke_async(
        f"Topic: {topic}\n\nAnalysis:\n{perspectives_text}\n\nConflicts:\n{conflicts}\n\n"
        "Provide your balanced recommendation."
    )

    return {
        "topic": topic,
        "perspectives": perspectives,
        "conflicts": str(conflicts),
        "recommendation": str(recommendation),
    }
```

---

### Example 3: Parallel Research Swarm

Research multiple sub-questions in parallel and synthesize into a structured report.

```python
async def parallel_research_swarm(main_question: str) -> str:
    # Step 1: Question decomposer
    decomposer = Agent(
        model=make_model(),
        system_prompt="Break a complex question into 4-6 specific, independent sub-questions as a JSON array of strings."
    )
    raw = await decomposer.invoke_async(
        f"Decompose into sub-questions: {main_question}\n"
        "Return: [\"sub-question 1\", \"sub-question 2\", ...]"
    )
    import json, re
    match = re.search(r'\[.*\]', str(raw), re.DOTALL)
    sub_questions = json.loads(match.group()) if match else [main_question]

    # Step 2: Parallel researchers — one per sub-question
    semaphore = asyncio.Semaphore(5)

    async def research_one(q: str) -> dict:
        async with semaphore:
            researcher = Agent(
                model=make_model(),
                system_prompt="Answer the question thoroughly with evidence and examples."
            )
            answer = await researcher.invoke_async(q)
            return {"question": q, "answer": str(answer)}

    findings = list(await asyncio.gather(*[research_one(q) for q in sub_questions]))

    # Step 3: Report writer
    report_writer = Agent(
        model=make_model(),
        system_prompt=(
            "Write a structured research report with: Executive Summary, "
            "Key Findings (one section per sub-question), Synthesis, and Conclusion."
        )
    )
    findings_text = "\n\n".join(
        f"Q: {f['question']}\nA: {f['answer']}" for f in findings
    )
    report = await report_writer.invoke_async(
        f"Main question: {main_question}\n\nResearch findings:\n\n{findings_text}"
    )
    return str(report)
```

---

## Performance Benchmarks and Cost Analysis

### Latency vs. Sequential

| Setup | Latency (approx.) | Notes |
|---|---|---|
| Sequential (N=5) | ~N * single_agent_latency | Fully serialized |
| Parallel swarm (N=5) | ~single_agent_latency | Bounded by slowest worker |
| Rate-limited (N=10, sem=3) | ~ceil(N/3) * latency | Controlled throughput |
| Pipeline (3 stages) | ~3 * latency | Each stage waits for previous |

Swarms trade cost for latency: 5 parallel agents finish in the time of 1 but consume 5x the tokens.

### Cost Estimation

```python
def estimate_swarm_cost(
    num_agents: int,
    input_tokens_per_agent: int,
    output_tokens_per_agent: int,
    input_price_per_1k: float = 0.00025,   # Claude Haiku approximate pricing
    output_price_per_1k: float = 0.00125,
) -> dict:
    total_input  = num_agents * input_tokens_per_agent
    total_output = num_agents * output_tokens_per_agent
    cost = (total_input / 1000 * input_price_per_1k +
            total_output / 1000 * output_price_per_1k)
    return {
        "total_input_tokens":  total_input,
        "total_output_tokens": total_output,
        "estimated_cost_usd":  round(cost, 6),
        "cost_vs_single":      f"{num_agents}x",
    }

# Example: 5-agent swarm, each sees 1000 input + produces 500 output tokens
print(estimate_swarm_cost(5, 1000, 500))
# {'total_input_tokens': 5000, 'total_output_tokens': 2500,
#  'estimated_cost_usd': 0.004375, 'cost_vs_single': '5x'}
```

### Optimization Strategies

- Use a cheap/fast model (Haiku) for worker agents; reserve powerful models (Sonnet/Opus) for the aggregator only.
- Keep worker system prompts short — they are included in every agent's input token count.
- Cap swarm size with `asyncio.Semaphore` to avoid Bedrock `ThrottlingException`.
- Use `memory_efficient_swarm` for large batches (> 50 items) to avoid OOM.
- Short-circuit: if the first K workers all agree, skip remaining workers (voting fast-path).

```python
async def fast_path_voting_swarm(question: str, threshold: float = 0.8) -> str:
    """Stop early once a supermajority is reached."""
    num_voters = 7
    semaphore = asyncio.Semaphore(num_voters)
    votes: list[str] = []
    done = asyncio.Event()

    async def vote(idx: int) -> None:
        async with semaphore:
            if done.is_set():
                return
            agent = Agent(
                model=make_model(),
                system_prompt="Answer with a single word: YES or NO."
            )
            result = str(await agent.invoke_async(question)).strip().upper()
            votes.append(result)
            # Check supermajority
            counted = len(votes)
            yes_frac = votes.count("YES") / counted
            no_frac  = votes.count("NO")  / counted
            if max(yes_frac, no_frac) >= threshold and counted >= 3:
                done.set()

    await asyncio.gather(*[vote(i) for i in range(num_voters)])
    winner = max(set(votes), key=votes.count)
    return f"{winner} ({votes.count(winner)}/{len(votes)} votes)"
```

---

## Testing Swarms

### Unit Testing Individual Workers

```python
import pytest

@pytest.mark.asyncio
async def test_single_worker_response():
    agent = Agent(
        model=BedrockModel(
            model_id="us.anthropic.claude-haiku-4-5-20251001-v1:0",
            region_name="us-east-1",
        ),
        system_prompt="Reply with exactly one word: 'OK'."
    )
    result = await agent.invoke_async("Are you ready?")
    assert "ok" in str(result).lower()
```

### Mocking Agents for Swarm Logic Tests

```python
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_voting_swarm_majority():
    mock_responses = ["YES", "YES", "NO", "YES", "NO"]

    async def mock_invoke(prompt):
        return mock_responses.pop(0)

    with patch("strands.Agent") as MockAgent:
        instance = MockAgent.return_value
        instance.invoke_async = AsyncMock(side_effect=mock_invoke)
        # Inject mock agents into your swarm function
        agents = [MockAgent() for _ in range(5)]
        results = await asyncio.gather(*[a.invoke_async("test") for a in agents])
        votes = [str(r) for r in results]
        winner = max(set(votes), key=votes.count)
        assert winner == "YES"
```

### Integration Test: Partial Failure

```python
@pytest.mark.asyncio
async def test_resilient_swarm_tolerates_failures():
    call_count = 0

    async def flaky_invoke(prompt):
        nonlocal call_count
        call_count += 1
        if call_count % 2 == 0:
            raise RuntimeError("Simulated failure")
        return f"OK-{call_count}"

    with patch("strands.Agent") as MockAgent:
        instance = MockAgent.return_value
        instance.invoke_async = AsyncMock(side_effect=flaky_invoke)
        agents = [MockAgent() for _ in range(4)]
        results = await resilient_swarm("test", agents, min_successes=2)
        assert len(results) >= 2
```

### Load / Throughput Test

```python
import time

async def benchmark_swarm(n_workers: int, prompt: str) -> dict:
    start = time.perf_counter()
    agents = [Agent(model=make_model(), system_prompt="Be brief.") for _ in range(n_workers)]
    results = await asyncio.gather(*[a.invoke_async(prompt) for a in agents])
    elapsed = time.perf_counter() - start
    return {
        "n_workers": n_workers,
        "total_seconds": round(elapsed, 2),
        "throughput_per_sec": round(n_workers / elapsed, 2),
        "all_succeeded": all(r is not None for r in results),
    }
```

---

## Concurrency Limit (Avoid Rate Limits)

```python
async def limited_swarm(items: list[str], max_concurrent: int = 5) -> list[str]:
    semaphore = asyncio.Semaphore(max_concurrent)

    async def run_one(item: str) -> str:
        async with semaphore:
            agent = Agent(model=make_model())
            return str(await agent.invoke_async(item))

    return list(await asyncio.gather(*[run_one(item) for item in items]))
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Sharing one Agent across concurrent calls | `Agent` is not thread/async-safe for concurrent calls | Create a **new `Agent` instance per swarm worker** |
| `ThrottlingException` from Bedrock | Too many concurrent API calls | Use `asyncio.Semaphore(3-5)` to cap concurrency |
| Results out of order | Unexpected — `asyncio.gather` preserves input order | Verify you are not using `as_completed` where order matters |
| Memory spike with large swarms | All results held in memory simultaneously | Use `memory_efficient_swarm` with chunked batches |
| Swarm hangs indefinitely | One worker deadlocked or awaiting a slow external resource | Set `asyncio.wait_for(coro, timeout=30)` per worker |
| Synthesis agent context overflow | Too many worker outputs concatenated | Truncate each worker output before passing to aggregator |
| Inconsistent voting results | Workers influenced by previous conversation context | Create fresh `Agent` instances; do not reuse across rounds |
| `ThrottlingException` in mesh (many rounds) | High total call volume | Add `asyncio.sleep(1)` between mesh rounds; reduce agent count |
| Tool conflicts in shared-tool swarms | Tool with mutable state accessed concurrently | Add `asyncio.Lock` inside tool, or use stateless tools only |
| High cost on large swarms | N agents * N tokens per call | Use Haiku for workers; Sonnet/Opus for aggregator only |
| Pipeline stage bottleneck | Middle stage has a single slow agent | Parallelise the bottleneck stage or increase its concurrency |

### Timeout Wrapper

```python
async def run_with_timeout(agent: Agent, prompt: str, timeout: float = 30.0) -> str | None:
    try:
        result = await asyncio.wait_for(agent.invoke_async(prompt), timeout=timeout)
        return str(result)
    except asyncio.TimeoutError:
        logger.warning("Agent timed out after %.1fs for prompt: %.60s...", timeout, prompt)
        return None
```

### Debug: Log Per-Worker Timing

```python
async def timed_swarm(items: list[str]) -> list[dict]:
    import time

    async def run_one(item: str, idx: int) -> dict:
        t0 = time.perf_counter()
        try:
            agent = Agent(model=make_model(), system_prompt="Be concise.")
            result = str(await agent.invoke_async(item))
            status = "ok"
        except Exception as exc:
            result = str(exc)
            status = "error"
        elapsed = time.perf_counter() - t0
        return {"idx": idx, "status": status, "elapsed": round(elapsed, 2), "result": result}

    return list(await asyncio.gather(*[run_one(item, i) for i, item in enumerate(items)]))
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Sharing one Agent across concurrent calls | Create a new Agent per swarm worker |
| `ThrottlingException` from Bedrock | Use `Semaphore` to cap concurrency to 3-5 |
| Results out of order | `asyncio.gather` preserves input order |
| Memory spike with large swarms | Process in batches of N using `limited_swarm` |
| No error isolation | Wrap each worker in `try/except`; use `resilient_swarm` |
| Forgetting to truncate inputs | Long documents can overflow context — chunk first |
| Coupling mesh rounds too tightly | Each round inflates context; cap rounds at 2-3 |
| Using blocking I/O inside async workers | Use `asyncio.to_thread(blocking_fn, ...)` for file/DB I/O |
