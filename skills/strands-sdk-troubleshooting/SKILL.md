---
name: strands-sdk-troubleshooting
description: "Use when debugging Strands SDK errors, fixing AccessDeniedException or ValidationException from Bedrock, resolving ModuleNotFoundError, agent not calling tools, empty responses, ThrottlingException, context limit exceeded, memory leaks, slow responses, tool call inspection, conversation state issues, multi-agent tracing, network errors, or any unexpected agent behavior. Triggers on: error, exception, not working, debug, AccessDeniedException, ValidationException, ThrottlingException, ModuleNotFoundError, agent broken, slow, latency, memory, leak, tool not called, state, trace, network, region, version."
---

# Troubleshooting — Strands SDK

## Diagnostic Checklist

Run through every item before diving into specific errors. Most issues are caught here.

- [ ] **Package installed?** `pip show strands-agents` — version present and not missing
- [ ] **Correct venv?** `which python` and `which pip` — both should point inside your venv
- [ ] **AWS identity** `aws sts get-caller-identity` — must show `arn:aws:iam::ACCOUNT:user/NAME` (not a role/SSO)
- [ ] **No SSO env vars** `echo $AWS_PROFILE $AWS_SESSION_TOKEN` — both should be empty for IAM-user auth
- [ ] **Region set?** `echo $AWS_REGION` — must match the region where your Bedrock model is enabled
- [ ] **Model ID exact match?** Copy the `modelId` string from `aws bedrock list-foundation-models` — do not guess
- [ ] **Model enabled in console?** Go to Bedrock console > Model access > confirm the model is "Access granted"
- [ ] **IAM policy attached?** Confirm `bedrock:InvokeModel` is in the user/role's policy for the specific model ARN
- [ ] **Dependencies installed?** `pip install -r requirements.txt` and check for version conflicts with `pip check`
- [ ] **Python version compatible?** `python --version` — strands-agents requires Python 3.10+
- [ ] **No firewall blocking Bedrock endpoint?** `curl -v https://bedrock-runtime.us-east-1.amazonaws.com` should reach the host
- [ ] **Tool docstrings present?** Every `@tool` function must have a docstring — missing docstrings silently prevent tool use
- [ ] **Conversation not contaminated?** `agent.conversation_manager.clear()` between unrelated sessions
- [ ] **Debug logging on?** Enable `logging.DEBUG` for `strands` before filing any bug report (see section below)

---

## Step-by-Step Debug Workflow

Use this flow to isolate the root cause systematically before trying fixes.

### Step 1 — Reproduce with a minimal script

Strip the problem down to the fewest lines that still reproduce it:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
logging.getLogger("strands").setLevel(logging.DEBUG)
logging.getLogger("botocore").setLevel(logging.WARNING)

from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)
agent = Agent(model=model)
result = agent("Say hello.")
print(str(result))
```

If this works, the bug is in your application code, not in Strands itself. Add complexity back one piece at a time.

### Step 2 — Confirm AWS identity and permissions

```bash
# Are you the expected identity?
aws sts get-caller-identity

# Can you call Bedrock at all?
aws bedrock list-foundation-models --region us-east-1 --query 'modelSummaries[?contains(modelId,`claude`)].modelId'

# Does this specific model respond?
aws bedrock-runtime invoke-model \
  --region us-east-1 \
  --model-id anthropic.claude-haiku-4-5-20251001-v1:0 \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":16,"messages":[{"role":"user","content":"hi"}]}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/response.json && cat /tmp/response.json
```

If `invoke-model` succeeds from the CLI but fails in Python, the issue is in credentials loading — not IAM policies.

### Step 3 — Isolate the failing component

```
Does the minimal script (Step 1) fail?
  YES → AWS credentials, model ID, network, or Strands install issue
  NO  → The bug is in how you build the agent or prompt

Does the CLI invoke-model (Step 2) fail?
  YES → IAM policy, model not enabled, region mismatch, or network
  NO  → boto3 / credentials loading issue in Python

Does tool use fail specifically?
  YES → See "Tool Call Debugging" section
  NO  → Model invocation is fine; check response parsing

Does the bug only appear after many turns?
  YES → Conversation history contamination or context window overflow
  NO  → Bug is in a single-turn path
```

### Step 4 — Inspect the full request/response

```python
from strands.hooks import AgentHook

class FullDumpHook(AgentHook):
    def on_llm_call(self, request):
        import json
        print("=== LLM REQUEST ===")
        print(json.dumps(request, indent=2, default=str))

    def on_llm_response(self, response):
        import json
        print("=== LLM RESPONSE ===")
        print(json.dumps(response, indent=2, default=str))

    def on_tool_use(self, tool_name, tool_input):
        print(f"=== TOOL CALL: {tool_name} ===")
        print(f"Input: {tool_input}")

    def on_tool_result(self, tool_name, tool_result):
        print(f"=== TOOL RESULT: {tool_name} ===")
        print(f"Result: {tool_result}")

    def on_error(self, error):
        import traceback
        print(f"=== ERROR: {type(error).__name__}: {error} ===")
        traceback.print_exc()

agent = Agent(model=model, tools=[...], hooks=[FullDumpHook()])
agent("your prompt")
```

### Step 5 — Binary search the system prompt

If removing the system prompt makes the bug disappear, the system prompt is the cause. Bisect it: cut it in half, test. Keep halving the failing section until you find the problematic instruction.

### Step 6 — Test with a different model

```python
# Swap to a different model to rule out model-specific issues
model = BedrockModel(model_id="anthropic.claude-sonnet-4-5-20251001-v1:0", region_name="us-east-1")
```

If the bug disappears with a different model, it is a model-specific behaviour, not a Strands bug.

---

## How to Read and Interpret Debug Logs

### Enable logging

```python
import logging

# Full Strands internals — use this first
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s %(name)s %(levelname)s %(message)s"
)
logging.getLogger("strands").setLevel(logging.DEBUG)

# Reduce boto3/botocore noise (keep at WARNING unless debugging AWS calls)
logging.getLogger("botocore").setLevel(logging.WARNING)
logging.getLogger("boto3").setLevel(logging.WARNING)
logging.getLogger("urllib3").setLevel(logging.WARNING)
```

### Log line anatomy

```
2025-01-15 10:23:01 strands.agent DEBUG  invoking model: messages=3 tools=2
2025-01-15 10:23:02 strands.agent DEBUG  model response: stop_reason=tool_use
2025-01-15 10:23:02 strands.agent DEBUG  tool_use: get_weather({"city": "Paris"})
2025-01-15 10:23:02 strands.agent DEBUG  tool_result: "Sunny, 22C in Paris"
2025-01-15 10:23:03 strands.agent DEBUG  model response: stop_reason=end_turn
```

Key fields to watch:
- `messages=N` — how many messages are in the current context window
- `stop_reason=tool_use` — model decided to call a tool (expected in agentic loops)
- `stop_reason=end_turn` — model finished generating (normal termination)
- `stop_reason=max_tokens` — response was cut off; increase `max_tokens`
- `stop_reason=stop_sequence` — a stop sequence fired early; check `stop_sequences=[]`

### Reading botocore logs for AWS errors

Set botocore to DEBUG only when diagnosing AWS-level failures:

```python
logging.getLogger("botocore").setLevel(logging.DEBUG)
```

Look for lines containing `HTTP Request` and `HTTP Response` to see the exact payload sent to Bedrock and the HTTP status code returned. A `400` with `ValidationException` means bad request body. A `403` means IAM denial.

---

## Full Error Reference

### Installation and Import Errors

#### `ModuleNotFoundError: No module named 'strands'`

```bash
pip install strands-agents

# If inside a venv and pip installs to the wrong place:
/path/to/venv/bin/pip install strands-agents

# Confirm the install landed in the right interpreter:
python -c "import strands; print(strands.__file__)"
```

#### `ModuleNotFoundError: No module named 'strands.models.anthropic'`

The Anthropic direct-API provider is an optional extra:

```bash
pip install "strands-agents[anthropic]"
```

#### `ModuleNotFoundError: No module named 'litellm'`

LiteLLM is optional:

```bash
pip install litellm
# For specific providers:
pip install "litellm[openai]"
pip install "litellm[google]"
```

#### `ImportError: cannot import name 'SlidingWindowConversationManager' from 'strands'`

Import path changed in newer versions:

```python
# Correct import (strands-agents >= 0.1.0):
from strands.conversation_manager import SlidingWindowConversationManager
```

#### `pip check` shows dependency conflicts

```bash
pip install --upgrade strands-agents
# Or pin to a known-good version:
pip install "strands-agents==0.1.x"
```

---

### AWS / Bedrock Errors

#### `AccessDeniedException: User is not authorized to perform: bedrock:InvokeModel`

Cause: IAM user is missing `bedrock:InvokeModel`, or an SSO role has an explicit Deny, or the model is not enabled in the account.

```bash
# Step 1 — unset all SSO / assumed-role credentials
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE

# Step 2 — set IAM user credentials
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_REGION=us-east-1

# Step 3 — confirm identity (must say user/NAME, not role/NAME)
aws sts get-caller-identity

# Step 4 — confirm Bedrock access
aws bedrock list-foundation-models --region us-east-1 > /dev/null && echo "OK"
```

Minimum IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-haiku-4-5-20251001-v1:0",
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-5-20251001-v1:0"
      ]
    }
  ]
}
```

For cross-region inference profiles, the resource pattern is different:

```json
"Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.*"
```

#### `AccessDeniedException: You don't have access to the model with the specified model ID`

The model exists but has not been enabled in your account. Fix: Bedrock console > Model access > Request access for that model > wait for approval (usually instant for Claude models).

#### `ValidationException: The model identifier provided is invalid`

```python
# WRONG — Anthropic API format, not Bedrock format
model_id = "claude-haiku-4-5-20251001"

# WRONG — missing version suffix
model_id = "anthropic.claude-haiku-4-5"

# CORRECT — Bedrock format
model_id = "anthropic.claude-haiku-4-5-20251001-v1:0"

# List exact available IDs:
import boto3
client = boto3.client("bedrock", region_name="us-east-1")
for m in client.list_foundation_models()["modelSummaries"]:
    if "claude" in m["modelId"]:
        print(m["modelId"])
```

#### `ValidationException: Invocation of model ID ... with on-demand throughput isn't supported`

The model requires a cross-region inference profile ID, not the bare model ID:

```python
# Use the inference profile ARN (global. or regional prefix):
model_id = "global.anthropic.claude-haiku-4-5-20251001-v1:0"
# OR the full ARN:
model_id = "arn:aws:bedrock:us-east-1:ACCOUNT:inference-profile/..."
```

#### `ThrottlingException: Too many requests`

```python
import asyncio
import time
from botocore.exceptions import ClientError

async def invoke_with_backoff(agent, prompt: str, max_attempts: int = 5) -> str:
    for attempt in range(max_attempts):
        try:
            return str(await agent.invoke_async(prompt))
        except ClientError as e:
            if e.response["Error"]["Code"] != "ThrottlingException":
                raise
            if attempt == max_attempts - 1:
                raise
            delay = (2 ** attempt) + (attempt * 0.1)  # jitter
            await asyncio.sleep(delay)
    raise RuntimeError("Max retry attempts exceeded")
```

For swarms, cap concurrency:

```python
semaphore = asyncio.Semaphore(3)  # max 3 concurrent Bedrock calls

async def rate_limited_invoke(agent, prompt):
    async with semaphore:
        return str(await agent.invoke_async(prompt))
```

#### `ServiceUnavailableException: The service is temporarily unavailable`

Bedrock regional outage or maintenance. Check https://health.aws.amazon.com/ and retry with exponential backoff. Consider failing over to a different region (see Region Availability Matrix).

#### `ModelTimeoutException: The model request timed out`

The model took too long for very long prompts or high load:

```python
model = BedrockModel(
    model_id="...",
    region_name="us-east-1",
    # Increase the boto3 read timeout:
    boto_client_config={"read_timeout": 300}
)
```

#### `ModelNotReadyException`

Model is being provisioned or in a cold-start state. Retry after a few seconds.

#### `ResourceNotFoundException: Could not find model`

Wrong region for this model. See the Region Availability Matrix below.

#### `ExpiredTokenException: The security token included in the request is expired`

Session token from SSO or `assume-role` expired:

```bash
# Refresh SSO session:
aws sso login --profile my-profile

# Or unset expired tokens and use long-term IAM credentials:
unset AWS_SESSION_TOKEN
```

#### `NoCredentialsError: Unable to locate credentials`

boto3 cannot find any credentials. Resolution order boto3 uses:
1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. `~/.aws/credentials` file
3. IAM instance profile (EC2/ECS/Lambda)
4. Container credentials

```bash
# Check what boto3 resolves:
python -c "import boto3; s = boto3.session.Session(); c = s.get_credentials(); print(c.method if c else 'NO CREDENTIALS')"
```

#### `EndpointResolutionError` / `Could not connect to the endpoint URL`

```bash
# Test Bedrock endpoint connectivity:
curl -v https://bedrock-runtime.us-east-1.amazonaws.com/

# If behind a proxy, set:
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
```

---

### Agent Behaviour Errors

#### Agent Invoked But Never Calls Tools

```python
# Diagnostic: print registered tools
print([t.__name__ for t in agent.tools])

# Diagnostic: check tool schema
from strands import tool
import inspect

@tool
def my_tool(x: str) -> str:
    """Does something."""
    return x

print(inspect.getdoc(my_tool))   # must be non-empty
print(my_tool.__name__)          # must be snake_case, descriptive
```

Root causes and fixes:
1. **Missing docstring** — the docstring IS the tool description the model reads. Without it, the model does not know what the tool does.
2. **Vague docstring** — "Does stuff" does not tell the model when to use the tool. Be specific: "Fetches live weather for a given city name using the OpenWeather API."
3. **System prompt conflicts** — a `"Do not use any external tools"` instruction overrides tool availability.
4. **Wrong prompt** — use explicit phrasing: `"Use the get_weather tool to check the weather in Paris."`
5. **Model temperature 0** — at temperature 0, the model is maximally conservative about tool use.
6. **Tool not in `tools=[]`** — verify `my_tool` is actually passed to `Agent(tools=[my_tool])`.

```python
# Force tool use via prompt:
result = agent(f"Use the {tool_func.__name__} tool to accomplish this task: {task}")
```

#### Empty or Truncated Response

```python
# Check stop_reason in the raw response via hook
class StopReasonHook(AgentHook):
    def on_llm_response(self, response):
        print(f"stop_reason: {response.get('stop_reason')}")
        print(f"output_tokens: {response.get('usage', {}).get('output_tokens')}")

# Fix 1: increase max_tokens
model = BedrockModel(model_id="...", max_tokens=8192)

# Fix 2: remove accidental stop sequences
model = BedrockModel(model_id="...", stop_sequences=[])

# Fix 3: check result access pattern — NOT str(result) which may show object repr
result = agent("prompt")
text = result.message           # preferred
text = str(result).strip()      # also works
```

#### `RuntimeError: no running event loop`

```python
# In a plain Python script — wrap with asyncio.run():
import asyncio
result = asyncio.run(agent.invoke_async("prompt"))

# In FastAPI (already async) — just await:
result = await agent.invoke_async("prompt")

# In Jupyter notebook — use nest_asyncio:
import nest_asyncio
nest_asyncio.apply()
result = asyncio.run(agent.invoke_async("prompt"))

# NEVER mix sync agent() call inside an async def — it will block the event loop:
# BAD:
async def handler():
    result = agent("prompt")   # blocks event loop
# GOOD:
async def handler():
    result = await agent.invoke_async("prompt")
```

#### Context / Token Limit Exceeded

```python
from strands.conversation_manager import SlidingWindowConversationManager

# Keep only the last N message pairs (user + assistant = 2 messages per turn)
manager = SlidingWindowConversationManager(window_size=20)
agent = Agent(model=model, conversation_manager=manager)

# Check current history length at any time:
messages = agent.conversation_manager.get_messages()
print(f"History length: {len(messages)} messages")
total_chars = sum(len(str(m)) for m in messages)
print(f"Approximate size: {total_chars} chars (~{total_chars // 4} tokens)")
```

#### Agent Loops Without Stopping (Infinite Tool Call Loop)

```python
# Symptom: agent keeps calling tools in circles without producing a final answer
# Fix 1: add max_turns to break the loop
result = agent("prompt", max_turns=10)

# Fix 2: make tool return values unambiguous — if a tool always returns
# "needs more info", the agent will keep trying. Return concrete results.

# Fix 3: inspect the loop with a hook
class LoopDetectorHook(AgentHook):
    def __init__(self):
        self.tool_calls = []

    def on_tool_use(self, tool_name, tool_input):
        self.tool_calls.append((tool_name, str(tool_input)))
        if len(self.tool_calls) > 5:
            # Check for repeated identical calls
            recent = self.tool_calls[-5:]
            if len(set(recent)) == 1:
                print(f"WARNING: tool {tool_name} called identically 5 times in a row")
```

#### Agent Gives Wrong Answer Repeatedly

```python
# Conversation history is contaminated from a previous session
agent.conversation_manager.clear()

# For multi-user scenarios, always clear before each user's session:
async def handle_user_request(prompt: str) -> str:
    agent.conversation_manager.clear()
    return str(await agent.invoke_async(prompt))
```

#### `str(result)` Shows Object Representation

```python
# BAD — prints <AgentResult object at 0x...>
print(result)

# GOOD
print(str(result))       # calls __str__ which returns the text
print(result.message)    # direct attribute access
text = str(result).strip()
```

---

## Tool Call Debugging

Inspect exactly what the agent sends to a tool and what it receives back.

### Dump every tool interaction

```python
from strands.hooks import AgentHook
import json

class ToolDebugHook(AgentHook):
    def on_tool_use(self, tool_name: str, tool_input: dict):
        print(f"\n{'='*50}")
        print(f"TOOL CALL: {tool_name}")
        print(f"INPUT:\n{json.dumps(tool_input, indent=2, default=str)}")

    def on_tool_result(self, tool_name: str, tool_result):
        print(f"RESULT from {tool_name}:")
        result_str = str(tool_result)
        # Truncate very long results for readability
        print(result_str[:500] + ("..." if len(result_str) > 500 else ""))
        print(f"{'='*50}\n")

    def on_error(self, error):
        if "tool" in str(error).lower():
            import traceback
            print(f"TOOL ERROR: {error}")
            traceback.print_exc()

agent = Agent(model=model, tools=[my_tool], hooks=[ToolDebugHook()])
```

### Validate tool schema before attaching

```python
from strands import tool
import inspect
import json

def inspect_tool_schema(tool_fn):
    """Print the schema that will be sent to the LLM for this tool."""
    print(f"Tool name:    {tool_fn.__name__}")
    print(f"Docstring:    {inspect.getdoc(tool_fn)}")
    sig = inspect.signature(tool_fn)
    print(f"Parameters:   {list(sig.parameters.keys())}")
    # Check type annotations are present
    for name, param in sig.parameters.items():
        if param.annotation is inspect.Parameter.empty:
            print(f"WARNING: parameter '{name}' has no type annotation — schema will be incomplete")

inspect_tool_schema(my_tool)
```

### Common tool call failure patterns

| Symptom | Root Cause | Fix |
|---|---|---|
| Tool called with `null` argument | Parameter not described in docstring | Add Args section to docstring |
| Tool called with wrong type | No type annotation on parameter | Add `param: str`, `param: int`, etc. |
| Tool called but result ignored | Tool returns None silently | Ensure tool always returns a string |
| Tool raises exception mid-run | Unhandled error inside tool | Wrap tool body in try/except and return error string |
| Tool name shown as `<lambda>` | Used lambda instead of named function | Define a proper `def` function with `@tool` |
| Tool not in model's context | Tool registered after Agent created | Always pass `tools=[]` at Agent construction time |

### Make tools fail gracefully

```python
@tool
def fetch_data(url: str) -> str:
    """Fetch data from a URL and return the response body.

    Args:
        url: The URL to fetch

    Returns:
        Response body text, or an error message if the fetch fails
    """
    try:
        import urllib.request
        with urllib.request.urlopen(url, timeout=10) as response:
            return response.read().decode("utf-8")[:4000]
    except Exception as e:
        # Return error as string — agent can decide what to do
        return f"Error fetching {url}: {type(e).__name__}: {e}"
```

---

## Conversation State Debugging

### Inspect current conversation history

```python
# See every message in the current context window
messages = agent.conversation_manager.get_messages()
for i, msg in enumerate(messages):
    role = msg.get("role", "?")
    content = msg.get("content", "")
    if isinstance(content, list):
        # Multi-part content (text + tool calls)
        text_parts = [c.get("text", "") for c in content if c.get("type") == "text"]
        content_preview = " | ".join(text_parts)[:100]
    else:
        content_preview = str(content)[:100]
    print(f"[{i}] {role}: {content_preview}")

print(f"\nTotal messages: {len(messages)}")
```

### Detect context window pressure

```python
def estimate_token_count(messages: list) -> int:
    """Rough estimate: 1 token ~ 4 characters."""
    total_chars = sum(len(str(m)) for m in messages)
    return total_chars // 4

messages = agent.conversation_manager.get_messages()
est_tokens = estimate_token_count(messages)
print(f"Estimated tokens in context: {est_tokens}")

# Claude Haiku context: 200k tokens
# Claude Sonnet context: 200k tokens
if est_tokens > 150_000:
    print("WARNING: approaching context limit — consider SlidingWindowConversationManager")
```

### Reset and seed conversation

```python
# Full reset
agent.conversation_manager.clear()

# Reset and inject specific history (e.g., restore from DB)
import json

# Save
history = agent.conversation_manager.get_messages()
json.dump(history, open("session.json", "w"))

# Restore in next session
history = json.load(open("session.json"))
agent = Agent(model=model, messages=history)
```

### Debug role sequencing errors

Bedrock requires messages to alternate user/assistant. If you manually build messages and get a `ValidationException` about message ordering:

```python
def validate_message_order(messages: list) -> list[str]:
    """Check for role sequencing violations."""
    issues = []
    for i in range(1, len(messages)):
        prev = messages[i-1].get("role")
        curr = messages[i].get("role")
        if prev == curr:
            issues.append(f"Messages {i-1} and {i} both have role='{curr}' — must alternate")
    return issues

issues = validate_message_order(agent.conversation_manager.get_messages())
for issue in issues:
    print(f"ISSUE: {issue}")
```

---

## Memory Leak Detection and Fixes

### Symptom: process memory grows unbounded over many requests

The most common cause is conversation history accumulating in a long-running process.

```python
import tracemalloc
import gc

# Enable memory tracing
tracemalloc.start()

# Run many agent invocations
for i in range(100):
    result = agent(f"Request {i}")
    # NOT clearing history — history grows with every call

# Take a snapshot
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics("lineno")
print("Top memory consumers:")
for stat in top_stats[:5]:
    print(stat)
```

### Fix 1: Use SlidingWindowConversationManager

```python
from strands.conversation_manager import SlidingWindowConversationManager

# Caps the history at N messages regardless of how many turns happen
manager = SlidingWindowConversationManager(window_size=20)
agent = Agent(model=model, conversation_manager=manager)
```

### Fix 2: Explicit clear between requests

```python
async def handle_request(prompt: str) -> str:
    agent.conversation_manager.clear()   # prevents unbounded growth
    return str(await agent.invoke_async(prompt))
```

### Fix 3: Create a fresh Agent per request (for truly stateless endpoints)

```python
# Model can be reused — it holds no per-request state
model = BedrockModel(model_id="...", region_name="us-east-1")

async def handle_request(prompt: str) -> str:
    # New Agent per request — zero cross-request memory leakage
    agent = Agent(model=model, system_prompt=SYSTEM_PROMPT)
    return str(await agent.invoke_async(prompt))
```

### Fix 4: Watch for large tool results accumulating in history

```python
# If a tool returns 50k characters of data per call, and you run 100 turns,
# you accumulate 5MB in conversation history.
# Truncate large tool results before they enter history:

@tool
def search_documents(query: str) -> str:
    """Search documents and return relevant excerpts."""
    results = do_search(query)
    # Cap result size — history-friendly
    return str(results)[:2000] + (f"\n[...truncated, {len(str(results))} total chars]"
                                   if len(str(results)) > 2000 else "")
```

---

## Performance Debugging (Slow Responses, High Latency)

### Measure latency per component

```python
import time
from strands.hooks import AgentHook

class LatencyHook(AgentHook):
    def __init__(self):
        self._llm_start = None
        self._tool_start = None
        self.llm_times = []
        self.tool_times = {}

    def on_llm_call(self, request):
        self._llm_start = time.perf_counter()

    def on_llm_response(self, response):
        if self._llm_start:
            elapsed = time.perf_counter() - self._llm_start
            self.llm_times.append(elapsed)
            tokens_out = response.get("usage", {}).get("output_tokens", 0)
            tps = tokens_out / elapsed if elapsed > 0 else 0
            print(f"LLM latency: {elapsed:.2f}s | output_tokens: {tokens_out} | {tps:.1f} tok/s")

    def on_tool_use(self, tool_name, tool_input):
        self._tool_start = time.perf_counter()

    def on_tool_result(self, tool_name, tool_result):
        if self._tool_start:
            elapsed = time.perf_counter() - self._tool_start
            self.tool_times.setdefault(tool_name, []).append(elapsed)
            print(f"Tool '{tool_name}' latency: {elapsed:.2f}s")

    def report(self):
        if self.llm_times:
            print(f"\nLLM: {len(self.llm_times)} calls, "
                  f"avg={sum(self.llm_times)/len(self.llm_times):.2f}s, "
                  f"max={max(self.llm_times):.2f}s")
        for tool, times in self.tool_times.items():
            print(f"Tool '{tool}': avg={sum(times)/len(times):.2f}s")

lat = LatencyHook()
agent = Agent(model=model, tools=[...], hooks=[lat])
agent("your prompt")
lat.report()
```

### Common performance issues and fixes

| Symptom | Cause | Fix |
|---|---|---|
| First call slow (5-10s), subsequent calls fast | Bedrock cold start | Pre-warm with a lightweight call at startup |
| All calls slow (~10s) in `ap-south-1` | Cross-region inference routing | Use `us-east-1` if latency is critical; measure both regions |
| Latency spikes every N calls | ThrottlingException causing retries | Cap concurrency with Semaphore; request quota increase |
| Tool calls dominate latency | External API calls inside tools are slow | Add caching inside tools; use async tools |
| Large context window slowing calls | Token processing overhead | Use SlidingWindowConversationManager to reduce context size |
| Streaming not reaching client | Not using `stream_async` | Use `async for event in agent.stream_async(prompt)` |

### Enable prompt caching to reduce latency and cost

```python
# Strands auto-adds cache_point for system prompts > 1024 tokens
# Cache hit saves both time (no re-processing) and cost (90% discount)
result = agent("prompt")
usage = result.usage
print(f"Cache read tokens: {usage.get('cache_read_input_tokens', 0)}")
print(f"Cache write tokens: {usage.get('cache_creation_input_tokens', 0)}")
# If cache_read > 0 on second call, caching is working
```

---

## Multi-Agent Debugging (Tracing Across Agents)

### Assign identifiers to each agent

```python
import uuid

def make_traced_agent(name: str, model, **kwargs) -> Agent:
    """Create an agent with a unique trace ID for log correlation."""
    trace_id = str(uuid.uuid4())[:8]

    class TracedHook(AgentHook):
        def on_llm_call(self, request):
            print(f"[{name}/{trace_id}] LLM call — messages: {len(request.get('messages', []))}")

        def on_tool_use(self, tool_name, tool_input):
            print(f"[{name}/{trace_id}] Tool: {tool_name}")

        def on_error(self, error):
            print(f"[{name}/{trace_id}] ERROR: {error}")

    return Agent(
        model=model,
        hooks=[TracedHook()],
        trace_attributes={"agent.name": name, "agent.trace_id": trace_id},
        **kwargs
    )

orchestrator = make_traced_agent("orchestrator", model, tools=[research, write_doc])
```

### Trace tool calls that invoke sub-agents

```python
from strands import tool

@tool
async def research(topic: str) -> str:
    """Research a topic using the specialist research agent.

    Args:
        topic: Topic to research

    Returns:
        Research findings
    """
    print(f"[research-subagent] invoked with topic='{topic}'")
    result = await research_agent.invoke_async(f"Research: {topic}")
    output = str(result)
    print(f"[research-subagent] returned {len(output)} chars")
    return output
```

### Debug A2A network calls

```python
# On the server side, log incoming requests:
from strands.multiagent.a2a import A2AServer

class DebugA2AServer(A2AServer):
    async def handle_request(self, request):
        print(f"[A2A server] incoming: {str(request)[:200]}")
        response = await super().handle_request(request)
        print(f"[A2A server] outgoing: {str(response)[:200]}")
        return response

# On the client side, test the connection before using it:
import httpx
r = httpx.get("http://localhost:8080/health")
print(f"A2A server health: {r.status_code}")
```

### Debug swarm results

```python
import asyncio
from strands import Agent
from strands.models import BedrockModel

async def debug_swarm(prompt: str) -> list[dict]:
    agents = [Agent(model=BedrockModel(...)) for _ in range(3)]
    results = await asyncio.gather(
        *[a.invoke_async(prompt) for a in agents],
        return_exceptions=True   # capture failures without aborting other agents
    )

    output = []
    for i, r in enumerate(results):
        if isinstance(r, Exception):
            print(f"Agent {i} FAILED: {type(r).__name__}: {r}")
            output.append({"agent": i, "status": "error", "error": str(r)})
        else:
            text = str(r)
            print(f"Agent {i} OK: {len(text)} chars")
            output.append({"agent": i, "status": "ok", "result": text})

    return output
```

### OpenTelemetry distributed tracing

```python
from strands import Agent
from strands.models import BedrockModel

# All agents in a multi-agent system should share the same service name
# and set span attributes for correlation
orchestrator = Agent(
    model=model,
    trace_attributes={
        "service.name": "my-multi-agent-system",
        "agent.role": "orchestrator",
        "deployment.environment": "production"
    }
)

sub_agent = Agent(
    model=model,
    trace_attributes={
        "service.name": "my-multi-agent-system",
        "agent.role": "researcher",
        "deployment.environment": "production"
    }
)

# Configure OTLP export via env vars:
# OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318
# OTEL_SERVICE_NAME=my-multi-agent-system
```

---

## Network / Connectivity Issues

### Diagnose connection failures

```bash
# Test basic HTTPS connectivity to Bedrock endpoint
curl -v https://bedrock-runtime.us-east-1.amazonaws.com/

# Expected: TCP connect succeeds, TLS handshake completes, HTTP 403 (unauthenticated is OK here)
# Problem: TCP timeout → firewall or VPN blocking
# Problem: TLS error → certificate issue, proxy interception

# Test DNS resolution
nslookup bedrock-runtime.us-east-1.amazonaws.com

# Test from within a container:
docker run --rm curlimages/curl -v https://bedrock-runtime.us-east-1.amazonaws.com/
```

### Corporate proxy configuration

```bash
# Set proxy for Python (boto3 respects these env vars):
export HTTPS_PROXY=http://proxy.corp.example.com:8080
export HTTP_PROXY=http://proxy.corp.example.com:8080
export NO_PROXY=localhost,127.0.0.1,169.254.169.254  # keep AWS metadata endpoint direct

# Or configure in boto3 directly:
import boto3
from botocore.config import Config

config = Config(proxies={"https": "http://proxy.corp.example.com:8080"})
session = boto3.session.Session()
client = session.client("bedrock-runtime", region_name="us-east-1", config=config)
```

### VPC endpoint (private connectivity)

If running in a VPC with a Bedrock VPC endpoint, ensure:
1. The VPC endpoint is enabled for `com.amazonaws.REGION.bedrock-runtime`
2. The security group allows outbound to the endpoint
3. The endpoint policy allows `bedrock:InvokeModel`
4. Private DNS is enabled on the VPC endpoint (so `bedrock-runtime.REGION.amazonaws.com` resolves to the private IP)

### SSL certificate errors

```bash
# Behind a corporate proxy with SSL inspection:
export AWS_CA_BUNDLE=/path/to/corporate-ca-bundle.pem
# OR
export REQUESTS_CA_BUNDLE=/path/to/corporate-ca-bundle.pem
```

---

## Region Availability Matrix

Not all Claude models are available in all AWS regions. Cross-region inference profiles (prefixed `global.`) route to the nearest available region automatically.

| Model | us-east-1 | us-west-2 | eu-west-1 | eu-central-1 | ap-southeast-1 | ap-northeast-1 | ap-south-1 |
|---|---|---|---|---|---|---|---|
| Claude Haiku 3.5 | Yes | Yes | Yes | Yes | Yes | Yes | Yes (via global.) |
| Claude Haiku 4.5 | Yes | Yes | Yes | No | No | No | Yes (via global.) |
| Claude Sonnet 3.5 | Yes | Yes | Yes | Yes | Yes | Yes | Yes (via global.) |
| Claude Sonnet 4.5 | Yes | Yes | Yes | No | No | No | Yes (via global.) |
| Claude Opus 4 | Yes | Yes | No | No | No | No | No |

**Cross-region inference profile IDs:**

```python
# Use these when your region does not have the model natively:
"global.anthropic.claude-haiku-4-5-20251001-v1:0"
"global.anthropic.claude-sonnet-4-5-20251001-v1:0"
"us.anthropic.claude-haiku-4-5-20251001-v1:0"   # US only
"eu.anthropic.claude-haiku-4-5-20251001-v1:0"   # EU only
```

Check current availability:

```bash
aws bedrock list-foundation-models \
  --region us-east-1 \
  --by-output-modality TEXT \
  --query 'modelSummaries[?contains(modelId,`claude`)].{id:modelId,status:modelLifecycle.status}' \
  --output table
```

---

## Version Compatibility Matrix

| strands-agents version | Python | Key features / changes |
|---|---|---|
| 0.1.x (initial) | 3.10+ | Agent, BedrockModel, @tool, SlidingWindowConversationManager, AgentHook |
| 0.1.x | 3.10+ | AnthropicModel (requires `[anthropic]` extra), LiteLLMModel (requires litellm) |
| 0.1.x | 3.10+ | A2AServer / AgentClient in `strands.multiagent.a2a` |
| 0.1.x | 3.10+ | `stream_async` generator interface |
| 0.1.x | 3.10+ | `trace_attributes` on Agent for OpenTelemetry |

Check installed version at any time:

```bash
pip show strands-agents
python -c "import strands; print(strands.__version__)"
```

Upgrade to latest:

```bash
pip install --upgrade strands-agents
```

Pin to a specific version (for reproducibility):

```bash
pip install "strands-agents==0.1.6"
# Lock all deps:
pip freeze > requirements-lock.txt
```

---

## Quick-Reference: Symptom to Fix

| Symptom | Most Likely Cause | Jump To |
|---|---|---|
| `ModuleNotFoundError: No module named 'strands'` | Package not installed or wrong venv | Error Reference > Installation |
| `AccessDeniedException` | SSO role or missing IAM policy | Error Reference > AWS / Bedrock |
| `ValidationException: model identifier` | Wrong model ID format | Error Reference > AWS / Bedrock |
| `ThrottlingException` | Too many concurrent Bedrock calls | Error Reference > AWS / Bedrock |
| `ServiceUnavailableException` | Regional Bedrock outage | Error Reference > AWS / Bedrock |
| `ModelTimeoutException` | Very long prompt, increase timeout | Error Reference > AWS / Bedrock |
| `ExpiredTokenException` | SSO session expired | Error Reference > AWS / Bedrock |
| `NoCredentialsError` | No AWS credentials found | Error Reference > AWS / Bedrock |
| Agent never calls tools | Missing docstring or vague description | Tool Call Debugging |
| Empty response | `max_tokens` too low or stop_sequence fired | Error Reference > Agent Behaviour |
| `RuntimeError: no running event loop` | Sync call inside async context | Error Reference > Agent Behaviour |
| Response truncated mid-sentence | `stop_reason=max_tokens` | Error Reference > Agent Behaviour |
| Agent loops infinitely | No termination condition in tool results | Error Reference > Agent Behaviour |
| `str(result)` shows object repr | Not calling `.message` or `str()` | Error Reference > Agent Behaviour |
| Memory grows unbounded | History not cleared between requests | Memory Leak Detection |
| High latency, slow first call | Cold start or large context | Performance Debugging |
| Tool args wrong type or null | Missing type hints or Args in docstring | Tool Call Debugging |
| Context limit exceeded | History too long | Conversation State Debugging |
| Multi-agent calls failing silently | Exception swallowed in sub-agent | Multi-Agent Debugging |
| A2A server unreachable | Port blocked or server not started | Network / Connectivity |
| `ValidationException` on role ordering | Malformed messages list | Conversation State Debugging |

---

## Community Resources and Where to Get Help

- **GitHub Issues:** https://github.com/strands-agents/sdk-python/issues — search before filing; many common errors have existing threads
- **GitHub Discussions:** https://github.com/strands-agents/sdk-python/discussions — ask questions, share patterns
- **Official docs:** https://strandsagents.com/docs
- **AWS Bedrock docs:** https://docs.aws.amazon.com/bedrock/latest/userguide/ — for IAM policies, model availability, quotas
- **Anthropic model docs:** https://docs.anthropic.com/claude/ — for model-specific parameters and limits
- **AWS re:Post (Bedrock tag):** https://repost.aws/tags/TAv_OB3iC9SE-Nm76g52Oqtw/amazon-bedrock
- **boto3 docs:** https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime.html

---

## How to File a Good Bug Report

A good bug report gets a fix in hours instead of weeks. Include all of the following.

### 1. Minimal reproduction script

Strip your code to the fewest lines that still reproduce the bug:

```python
# MINIMAL REPRODUCTION
import logging
logging.basicConfig(level=logging.DEBUG)

from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1")
agent = Agent(model=model)
result = agent("Hello")  # <-- THIS FAILS with: [paste exact error]
```

### 2. Exact error output

Copy the complete stack trace, not just the last line:

```
Traceback (most recent call last):
  File "repro.py", line 8, in <module>
    result = agent("Hello")
  File "/path/to/strands/agent.py", line 42, in __call__
    ...
botocore.exceptions.ClientError: An error occurred (AccessDeniedException) when calling
the InvokeModel operation: User: arn:aws:iam::123456789:user/myuser is not authorized to
perform: bedrock:InvokeModel on resource:
arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-haiku-4-5-20251001-v1:0
```

### 3. Environment details

```bash
# Run this block and paste the output:
python --version
pip show strands-agents | grep -E "Name|Version"
pip show boto3 botocore | grep -E "Name|Version"
python -c "import platform; print(platform.platform())"
aws --version
```

### 4. Steps to reproduce

Numbered, exact steps from a clean environment:
1. `python -m venv venv && source venv/bin/activate`
2. `pip install strands-agents==X.Y.Z`
3. `export AWS_REGION=us-east-1`
4. `python repro.py`
5. Expected: prints agent response
6. Actual: raises `AccessDeniedException`

### 5. What you have already tried

List the fixes you attempted so maintainers do not suggest them again.

### 6. Sanitise before posting

- Replace real AWS account IDs with `123456789012`
- Replace real API keys with `REDACTED`
- Replace internal hostnames/URLs with `example.internal`
- Keep the error message and model IDs intact — they are needed for diagnosis
