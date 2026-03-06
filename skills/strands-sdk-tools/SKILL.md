---
name: strands-sdk-tools
description: "Use when defining custom tools with @tool decorator, using Strands built-in tools, integrating MCP tools with an agent, passing tools to Agent constructor, or debugging tool call failures. Triggers on: @tool, use_aws, file_read, http_request, tool not found, MCP, tool schema, built-in tools."
---

# Tools — Strands SDK

## Overview

Tools are the mechanism by which an agent takes actions beyond text generation. A tool is a Python callable decorated with `@tool` (or a built-in / MCP-sourced tool object) that gets registered with the agent at construction time. During inference the LLM emits a `tool_use` content block; Strands intercepts it, dispatches to the matching Python function, injects the result back as a `tool_result` block, and continues generation. This loop repeats until the model stops emitting tool calls.

```
Agent("Do X")
  └─► LLM generates tool_use { name, input }
        └─► Strands dispatches to Python function
              └─► result injected as tool_result
                    └─► LLM continues → final text response
```

---

## Quick Reference

| Category | Import | Purpose |
|---|---|---|
| Custom tool | `from strands import tool` | `@tool` decorator on any function |
| Built-in: file read | `from strands.tools import file_read` | Read files from disk |
| Built-in: file write | `from strands.tools import file_write` | Write files to disk |
| Built-in: HTTP | `from strands.tools import http_request` | Make HTTP/HTTPS requests |
| Built-in: AWS | `from strands.tools import use_aws` | Call any AWS service via boto3 |
| Built-in: shell | `from strands.tools import shell` | Execute shell commands |
| Built-in: Python REPL | `from strands.tools import python_repl` | Execute arbitrary Python code |
| MCP tools | `from strands.tools.mcp import MCPClient` | Connect to an MCP tool server |

---

## How `@tool` Works Internally — Schema Generation

When Python imports a module containing `@tool`-decorated functions, the decorator performs the following at decoration time (not at call time):

1. **Docstring parsing** — The function's `__doc__` string is split into a short description (first paragraph) and an `Args:` section. Each `param: description` line under `Args:` becomes the per-property `description` in the JSON schema.

2. **Type-hint introspection** — `inspect.signature` and `typing.get_type_hints` are used to map each parameter's annotation to a JSON Schema type:
   - `str` → `{"type": "string"}`
   - `int` → `{"type": "integer"}`
   - `float` → `{"type": "number"}`
   - `bool` → `{"type": "boolean"}`
   - `list` / `List[T]` → `{"type": "array", "items": ...}`
   - `dict` / `Dict[str, T]` → `{"type": "object"}`
   - `Optional[T]` → type is unwrapped; the parameter is omitted from `required`

3. **Required vs optional** — Parameters without a default value are added to the `required` array. Parameters with a default (including `None`) are omitted from `required`.

4. **Tool spec object** — The decorator wraps the function in a `ToolSpec` (or equivalent) object with `.name`, `.description`, and `.input_schema` attributes that Strands passes verbatim to the model's API as a tool definition.

The JSON schema that the LLM sees looks like:

```json
{
  "name": "get_weather",
  "description": "Get current weather for a city.",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "Name of the city to get weather for"
      }
    },
    "required": ["city"]
  }
}
```

Because the schema is generated once at import time, the LLM always sees the schema as it existed when the module was loaded. Hot-reloading or modifying the function at runtime will not update the schema without re-importing.

---

## Custom Tool Pattern — Minimal Example

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

model = BedrockModel(model_id="us.anthropic.claude-3-5-haiku-20241022-v1:0")
agent = Agent(model=model, tools=[get_weather])
result = agent("What's the weather in Paris?")
print(result)
```

---

## Tool Schema Rules

- The function **docstring is mandatory** — it becomes the tool description sent to the LLM. Without it the decorator raises an error or produces a schema with an empty description, causing the model to ignore or misuse the tool.
- The `Args:` section in the docstring is the per-parameter description the LLM uses to fill in arguments correctly. Omitting it degrades model accuracy on complex tools.
- Type hints on every parameter are required for correct schema generation. Untyped parameters default to `{"type": "string"}` which may cause silent type coercion bugs.
- Return type annotation (`-> str`, `-> dict`, etc.) is not used in schema generation but documents intent and helps static analysis.
- Keep function names descriptive and in `snake_case`. The name is passed verbatim to the model; it affects how reliably the model chooses the tool.
- Avoid overly long docstrings. The description is included in every API call; excessively long descriptions inflate token usage.

---

## Async vs Sync Tool Patterns

Strands supports both synchronous and asynchronous tool functions. The agent runtime detects the function type via `asyncio.iscoroutinefunction` and awaits async tools automatically when running in an async context.

**Synchronous tool** (default, works everywhere):

```python
@tool
def fetch_price(ticker: str) -> str:
    """Fetch the latest stock price for a ticker symbol.

    Args:
        ticker: Stock ticker symbol, e.g. AAPL
    """
    import requests
    resp = requests.get(f"https://api.example.com/price/{ticker}", timeout=5)
    resp.raise_for_status()
    return f"{ticker}: ${resp.json()['price']:.2f}"
```

**Asynchronous tool** (use when the underlying I/O is async):

```python
import httpx
from strands import tool

@tool
async def fetch_price_async(ticker: str) -> str:
    """Fetch the latest stock price for a ticker symbol asynchronously.

    Args:
        ticker: Stock ticker symbol, e.g. AAPL
    """
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"https://api.example.com/price/{ticker}")
        resp.raise_for_status()
        data = resp.json()
    return f"{ticker}: ${data['price']:.2f}"

# Must use invoke_async when any tool is async
import asyncio
result = asyncio.run(agent.invoke_async("What's the price of AAPL?"))
```

**Rule of thumb:** if the agent itself is being called with `await agent.invoke_async(...)`, async tools are fine. If called synchronously via `agent(...)`, use sync tools only — mixing async tools into a sync call path will raise a `RuntimeError` about event loop nesting.

---

## Tools with Complex Input Types

### Lists

```python
from typing import List

@tool
def summarize_urls(urls: List[str], max_words: int = 200) -> str:
    """Fetch and summarize multiple URLs.

    Args:
        urls: List of URLs to fetch and summarize
        max_words: Maximum words in each summary, defaults to 200
    """
    results = []
    for url in urls:
        # ... fetch and summarize
        results.append(f"[{url}] summary here")
    return "\n".join(results)
```

Generated schema fragment:

```json
"urls": {"type": "array", "items": {"type": "string"}, "description": "List of URLs..."},
"max_words": {"type": "integer", "description": "Maximum words..."}
```

`max_words` is absent from `required` because it has a default value.

### Dicts / Structured objects

```python
from typing import Dict, Any

@tool
def create_user(profile: Dict[str, Any]) -> str:
    """Create a new user account from a profile dict.

    Args:
        profile: User profile with keys: name (str), email (str), role (str)
    """
    name = profile.get("name", "Unknown")
    email = profile.get("email")
    role = profile.get("role", "viewer")
    if not email:
        return "Error: email is required"
    # ... persist
    return f"Created user {name} ({email}) with role {role}"
```

Tip: when the dict has a well-known shape, describe its expected keys explicitly in the `Args:` docstring. The model uses this description to construct the dict correctly.

### Optional parameters

```python
from typing import Optional

@tool
def search_docs(
    query: str,
    limit: int = 10,
    language: Optional[str] = None,
) -> str:
    """Search the documentation index.

    Args:
        query: Full-text search query
        limit: Maximum number of results to return, defaults to 10
        language: Filter by programming language, e.g. python. Omit for all languages.
    """
    filters = {}
    if language:
        filters["language"] = language
    # ... perform search
    return f"Found results for '{query}' (limit={limit}, lang={language})"
```

`Optional[str]` unwraps to `"type": "string"` in the schema and is excluded from `required`. The model may omit the argument entirely, in which case Python receives the default `None`.

---

## Tool Result Formatting Best Practices

The return value of a tool function is converted to a string and injected as the `tool_result` content. Follow these guidelines:

- **Return plain text for simple answers.** The model reads the result as-is.

  ```python
  return "The file has 42 lines."
  ```

- **Return structured text (not raw dicts) for complex results.** The model handles JSON strings better than Python repr output.

  ```python
  import json
  return json.dumps({"status": "ok", "records": records}, indent=2)
  ```

- **Include units and context.** `"42"` is ambiguous; `"42 lines in /tmp/data.csv"` is not.

- **Keep results concise.** Very large results (thousands of lines) fill the context window and degrade later reasoning. Truncate or paginate:

  ```python
  lines = content.split("\n")
  if len(lines) > 100:
      lines = lines[:100]
      lines.append(f"... ({len(content.split(chr(10))) - 100} more lines truncated)")
  return "\n".join(lines)
  ```

- **Distinguish success from failure clearly.** Start error results with `"Error:"` so the model can decide whether to retry or report back:

  ```python
  try:
      result = do_thing()
      return f"Success: {result}"
  except Exception as e:
      return f"Error: {e}"
  ```

- **Never return `None`.** If the tool has nothing meaningful to return, return an explicit confirmation string: `"Done."` or `"No results found."`.

---

## Passing Context and State to Tools

### Closures (recommended for simple shared state)

Closures let you capture configuration or lightweight objects at definition time without global variables:

```python
def make_db_tools(connection_string: str):
    import sqlite3
    conn = sqlite3.connect(connection_string)

    @tool
    def query_db(sql: str) -> str:
        """Execute a read-only SQL query against the database.

        Args:
            sql: SQL SELECT statement to execute
        """
        cursor = conn.execute(sql)
        rows = cursor.fetchmany(50)
        return "\n".join(str(r) for r in rows) or "No rows returned."

    @tool
    def insert_record(table: str, data: str) -> str:
        """Insert a JSON-encoded record into a table.

        Args:
            table: Target table name
            data: JSON string with column-value pairs
        """
        import json
        record = json.loads(data)
        cols = ", ".join(record.keys())
        placeholders = ", ".join("?" * len(record))
        conn.execute(f"INSERT INTO {table} ({cols}) VALUES ({placeholders})", list(record.values()))
        conn.commit()
        return f"Inserted 1 row into {table}."

    return [query_db, insert_record]

tools = make_db_tools("/data/app.db")
agent = Agent(model=model, tools=tools)
```

### Classes (recommended for stateful or lifecycle-managed tools)

```python
class S3Tools:
    def __init__(self, bucket: str):
        import boto3
        self.bucket = bucket
        self.s3 = boto3.client("s3")

    @tool
    def upload_file(self, local_path: str, s3_key: str) -> str:
        """Upload a local file to S3.

        Args:
            local_path: Absolute path to the local file
            s3_key: Destination key in the S3 bucket
        """
        self.s3.upload_file(local_path, self.bucket, s3_key)
        return f"Uploaded {local_path} to s3://{self.bucket}/{s3_key}"

    @tool
    def list_objects(self, prefix: str = "") -> str:
        """List objects in the S3 bucket.

        Args:
            prefix: Key prefix to filter results, e.g. 'logs/'
        """
        resp = self.s3.list_objects_v2(Bucket=self.bucket, Prefix=prefix)
        keys = [o["Key"] for o in resp.get("Contents", [])]
        return "\n".join(keys) if keys else "No objects found."

s3 = S3Tools(bucket="my-data-bucket")
agent = Agent(model=model, tools=[s3.upload_file, s3.list_objects])
```

Note: when passing bound methods, `self` is already bound and is not part of the tool schema — the LLM never sees or provides the `self` argument.

---

## Error Handling Inside Tools

### Return errors as strings (preferred)

For expected, recoverable errors (bad input, resource not found, API 4xx), return an error string. This lets the model reason about the failure and try a different approach:

```python
@tool
def read_config(path: str) -> str:
    """Read a configuration file and return its contents.

    Args:
        path: Absolute path to the config file
    """
    import os
    if not os.path.isabs(path):
        return "Error: path must be absolute, e.g. /etc/app/config.yaml"
    if not os.path.exists(path):
        return f"Error: file not found at {path}"
    try:
        with open(path) as f:
            return f.read()
    except PermissionError:
        return f"Error: permission denied reading {path}"
```

### Raise exceptions (for unexpected, unrecoverable errors)

For bugs or infrastructure failures that the LLM cannot fix by retrying with different arguments, raising an exception is appropriate. Strands will catch it and surface it as an error in the agent's response:

```python
@tool
def critical_operation(payload: str) -> str:
    """Perform a critical database write.

    Args:
        payload: JSON-encoded operation payload
    """
    import json
    data = json.loads(payload)          # May raise JSONDecodeError — let it propagate
    result = db.execute_critical(data)  # Infrastructure failure — let it propagate
    return f"Operation completed: {result}"
```

### Validation pattern

```python
@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email message.

    Args:
        to: Recipient email address
        subject: Email subject line
        body: Plain-text email body
    """
    import re
    if not re.match(r"[^@]+@[^@]+\.[^@]+", to):
        return f"Error: '{to}' is not a valid email address"
    if not subject.strip():
        return "Error: subject cannot be empty"
    # ... send
    return f"Email sent to {to} with subject '{subject}'"
```

---

## Tool Chaining Patterns

One tool calling another is straightforward — they are plain Python functions. The `@tool` decorator does not prevent direct calls.

### Direct call (simple pipeline)

```python
@tool
def fetch_raw(url: str) -> str:
    """Fetch raw HTML from a URL.

    Args:
        url: The URL to fetch
    """
    import requests
    return requests.get(url, timeout=10).text

@tool
def extract_title(url: str) -> str:
    """Fetch a URL and extract its HTML title tag.

    Args:
        url: The URL to fetch and parse
    """
    from html.parser import HTMLParser

    html = fetch_raw.func(url)  # Call the underlying function directly

    class TitleParser(HTMLParser):
        title = ""
        _in_title = False
        def handle_starttag(self, tag, attrs):
            if tag == "title": self._in_title = True
        def handle_data(self, data):
            if self._in_title: self.title += data
        def handle_endtag(self, tag):
            if tag == "title": self._in_title = False

    p = TitleParser()
    p.feed(html)
    return p.title.strip() or "No title found"
```

`fetch_raw.func` accesses the original unwrapped function to avoid double-registering the tool call in the agent's trace.

### Agent as orchestrator (multi-agent chaining)

```python
from strands import Agent

research_agent = Agent(model=model, tools=[web_search, fetch_raw])
write_agent = Agent(model=model, tools=[file_write])

@tool
def research_and_write(topic: str, output_path: str) -> str:
    """Research a topic and write a report to disk.

    Args:
        topic: Topic to research
        output_path: Absolute file path for the output report
    """
    research = str(research_agent(f"Research {topic} and return a detailed summary."))
    str(write_agent(f"Write this report to {output_path}:\n\n{research}"))
    return f"Report written to {output_path}"
```

---

## Built-in Tools — Full Usage Examples

### `file_read`

Reads the contents of a file from the local filesystem. The agent provides the `path` argument.

```python
from strands.tools import file_read
from strands import Agent

agent = Agent(model=model, tools=[file_read])
result = agent("Read /etc/hostname and tell me the machine name.")
```

The tool returns the file contents as a string. For binary files it returns a base64-encoded representation. On permission or not-found errors it returns an error string.

### `file_write`

Writes content to a file. Creates parent directories if needed.

```python
from strands.tools import file_write

agent = Agent(model=model, tools=[file_write])
result = agent("Create /tmp/hello.txt with the text 'Hello, World!'")
```

The agent provides `path` and `content` arguments. Returns a confirmation string on success.

### `http_request`

Makes HTTP requests. Supports GET, POST, PUT, DELETE. Handles JSON and form bodies.

```python
from strands.tools import http_request

agent = Agent(model=model, tools=[http_request])

# The agent will call http_request with method, url, headers, body as needed:
result = agent("POST to https://httpbin.org/post with JSON body {\"key\": \"value\"} and print the response.")
```

The tool returns the response body as a string. For non-2xx responses it returns the status code and body so the model can react.

### `shell`

Executes a shell command in a subprocess and returns stdout + stderr.

```python
from strands.tools import shell

agent = Agent(model=model, tools=[shell])
result = agent("Run `df -h` and summarize disk usage.")
```

Security note: only give `shell` access to agents in trusted environments. The model can construct arbitrary shell commands.

```python
# Restrict by wrapping shell in a safer custom tool:
@tool
def safe_shell(command: str) -> str:
    """Run a pre-approved read-only shell command.

    Args:
        command: Shell command to execute (read-only commands only)
    """
    ALLOWED_PREFIXES = ("ls ", "df ", "du ", "cat ", "echo ", "pwd", "date")
    if not any(command.startswith(p) for p in ALLOWED_PREFIXES):
        return f"Error: command '{command}' is not in the allowed list"
    import subprocess
    proc = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=10)
    return proc.stdout + proc.stderr
```

### `use_aws`

Calls any AWS service method via boto3. The agent provides `service`, `operation`, and `parameters`.

```python
from strands.tools import use_aws

agent = Agent(model=model, tools=[use_aws])

# List S3 buckets:
result = agent("List all my S3 buckets.")

# Describe EC2 instances:
result = agent("List all running EC2 instances in us-east-1.")

# Put an item into DynamoDB:
result = agent(
    "Put an item into DynamoDB table 'Users' with PK='user#123' and attributes name='Alice', age=30."
)
```

`use_aws` requires valid AWS credentials in the environment (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, optionally `AWS_SESSION_TOKEN`, and `AWS_DEFAULT_REGION` or `AWS_REGION`). It returns the boto3 response dict serialized as a JSON string.

### `python_repl`

Executes arbitrary Python code in an isolated namespace and returns stdout output and the repr of the last expression.

```python
from strands.tools import python_repl

agent = Agent(model=model, tools=[python_repl])
result = agent("Calculate the first 10 Fibonacci numbers using Python and show the list.")
```

The namespace persists within a single agent session, so variables defined in one call are available in subsequent calls. Use this for data analysis, computations, and prototyping.

```python
# Combining python_repl with file_read for data analysis:
agent = Agent(model=model, tools=[file_read, python_repl])
result = agent("Read /data/sales.csv, then use Python to compute the total revenue and top 5 products.")
```

---

## MCP Tool Integration

MCP (Model Context Protocol) servers expose tools over a JSON-RPC interface. Strands connects to them via `MCPClient`.

### Stdio transport (subprocess-based MCP server)

```python
import asyncio
from strands import Agent
from strands.tools.mcp import MCPClient
from strands.models import BedrockModel

model = BedrockModel(model_id="us.anthropic.claude-3-5-haiku-20241022-v1:0")

async def main():
    async with MCPClient(
        {"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]}
    ) as mcp:
        tools = await mcp.list_tools_async()
        agent = Agent(model=model, tools=tools)
        result = await agent.invoke_async("List all files in /tmp and tell me the 3 largest.")
        print(result)

asyncio.run(main())
```

### SSE transport (HTTP-based MCP server)

```python
async with MCPClient({"url": "http://localhost:3001/sse"}) as mcp:
    tools = await mcp.list_tools_async()
    agent = Agent(model=model, tools=tools)
    result = await agent.invoke_async("Use the available tools to answer my question.")
```

### MCP with authentication

For MCP servers that require authentication, pass headers via the transport config:

```python
async with MCPClient({
    "url": "https://my-mcp-server.example.com/sse",
    "headers": {
        "Authorization": f"Bearer {api_token}",
        "X-Workspace-ID": workspace_id,
    }
}) as mcp:
    tools = await mcp.list_tools_async()
    agent = Agent(model=model, tools=tools)
```

For stdio-based servers that authenticate via environment variables:

```python
import os

async with MCPClient({
    "command": "npx",
    "args": ["-y", "@company/mcp-server"],
    "env": {**os.environ, "MCP_API_KEY": api_key, "MCP_TENANT": tenant_id}
}) as mcp:
    tools = await mcp.list_tools_async()
```

### Mixing MCP tools with custom tools

```python
async with MCPClient({"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/data"]}) as mcp:
    mcp_tools = await mcp.list_tools_async()

    @tool
    def summarize_text(text: str, max_sentences: int = 3) -> str:
        """Summarize a block of text to a given number of sentences.

        Args:
            text: The text to summarize
            max_sentences: Maximum number of sentences in the summary
        """
        sentences = text.split(". ")
        return ". ".join(sentences[:max_sentences]) + ("." if len(sentences) > max_sentences else "")

    agent = Agent(model=model, tools=mcp_tools + [summarize_text])
    result = await agent.invoke_async("Read /data/report.txt and give me a 2-sentence summary.")
```

---

## Debugging Tool Calls

### Enable verbose logging

Set the `STRANDS_LOG_LEVEL` environment variable or configure Python logging before creating the agent:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
# or for just Strands:
logging.getLogger("strands").setLevel(logging.DEBUG)
```

This logs every tool dispatch: the tool name, the input arguments the model provided, and the result returned.

### Inspect the raw tool call arguments

Wrap any tool to print what the model actually sends:

```python
from strands import tool
import json

def debug_wrap(fn):
    original = fn.func if hasattr(fn, "func") else fn

    @tool
    def wrapper(**kwargs) -> str:
        print(f"[DEBUG] Tool '{fn.name}' called with: {json.dumps(kwargs, indent=2)}")
        result = original(**kwargs)
        print(f"[DEBUG] Tool '{fn.name}' returned: {result!r}")
        return result

    wrapper.__name__ = fn.name
    return wrapper

wrapped_tool = debug_wrap(my_tool)
agent = Agent(model=model, tools=[wrapped_tool])
```

### Check the agent's event stream

Strands agents expose an event stream that includes `tool_use` and `tool_result` events:

```python
for event in agent.stream("Do something with tools"):
    if event.get("type") == "tool_use":
        print(f"Tool called: {event['name']}, input: {event['input']}")
    elif event.get("type") == "tool_result":
        print(f"Tool result: {event['content']}")
    elif event.get("type") == "text":
        print(event["text"], end="", flush=True)
```

### Verify schema generation

Inspect the schema your `@tool` function generates before running the agent:

```python
from strands import tool

@tool
def my_tool(x: int, label: str = "default") -> str:
    """My tool description.

    Args:
        x: Integer input
        label: Optional label string
    """
    return f"{label}: {x}"

import json
print(json.dumps(my_tool.tool_spec, indent=2))
# Inspect: are required fields correct? Are descriptions present?
```

---

## Performance Tips

- **Avoid heavy computation in tool bodies.** The model waits synchronously for tool results. Long-running tools (>5s) increase total response latency linearly with the number of tool calls. Offload heavy work to background tasks and return a job ID if needed.

- **Cache expensive lookups.** If a tool fetches from an external API and is likely to be called multiple times with the same input, use `functools.lru_cache` or a dict cache:

  ```python
  from functools import lru_cache

  @lru_cache(maxsize=256)
  def _fetch_ticker_price(ticker: str) -> float:
      import requests
      return requests.get(f"https://api.example.com/price/{ticker}").json()["price"]

  @tool
  def get_stock_price(ticker: str) -> str:
      """Get the current stock price for a ticker.

      Args:
          ticker: Stock ticker symbol
      """
      price = _fetch_ticker_price(ticker.upper())
      return f"{ticker.upper()}: ${price:.2f}"
  ```

- **Return only what is needed.** If a tool returns a 10 MB JSON blob, every subsequent LLM call in the same session carries that in the context. Summarize or extract only the relevant fields before returning.

- **Batch operations when possible.** Instead of a tool that looks up one record, prefer a tool that accepts a list and returns multiple results in one call. This reduces the number of model round-trips.

  ```python
  @tool
  def batch_lookup(ids: List[str]) -> str:
      """Look up multiple records by ID in a single call.

      Args:
          ids: List of record IDs to look up
      """
      results = db.batch_get(ids)
      return json.dumps(results)
  ```

- **Use async tools for I/O-bound operations.** Network calls (HTTP, database, AWS) release the event loop while waiting, allowing the Python process to handle other coroutines. This matters when multiple agents run concurrently.

- **Pre-warm connections.** Establish database connections, boto3 clients, or HTTP sessions outside the tool function (via closures or class `__init__`) so the tool body does not pay connection setup latency on every call.

---

## Common Mistakes and Troubleshooting

| Mistake | Symptom | Fix |
|---|---|---|
| Missing docstring on `@tool` function | Tool has empty description; model ignores it or uses it incorrectly | Add a docstring with at least a one-line description |
| Missing `Args:` section in docstring | Model guesses parameter meanings; accuracy drops on multi-param tools | Add an `Args:` section with one line per parameter |
| No type hints on parameters | Schema defaults all params to `string`; numeric/boolean/list args silently coerced | Annotate every parameter with its correct type |
| Tool not called at all | Agent responds without invoking any tool | Rephrase the prompt to explicitly mention what the tool does; verify the tool is in the `tools=` list |
| `use_aws` raises `NoCredentialsError` | AWS credentials not found | Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION` in the environment |
| Async tool in sync agent | `RuntimeError: This event loop is already running` | Use `agent.invoke_async` + `asyncio.run`, or convert the tool to sync |
| MCP `list_tools_async` returns empty list | MCP server failed to start or wrong command | Check that `npx` / the server binary is installed; run the command manually to see errors |
| Tool result too large | Context window exceeded after a few turns | Truncate or paginate results inside the tool function |
| `tool.func` attribute missing | Calling `tool.func(...)` raises `AttributeError` | Use `tool.__wrapped__` or call the original function directly by its name before decoration |
| `@tool` on a method without binding | `self` appears in the LLM schema as a required parameter | Pass bound methods (`instance.method`) to `tools=`, not unbound class methods |
| `Optional[str]` treated as required | Model always provides the optional argument | Ensure a default value is set: `param: Optional[str] = None` |
| Returning `None` from a tool | Model receives `"None"` string; may interpret as "no data found" and hallucinate | Always return an explicit string: `"Done."`, `"No results."`, or a JSON payload |
