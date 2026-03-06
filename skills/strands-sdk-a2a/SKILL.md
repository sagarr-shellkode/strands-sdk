---
name: strands-sdk-a2a
description: "Use when implementing Agent-to-Agent (A2A) communication, having one Strands agent call another agent over HTTP, setting up A2AServer or AgentClient, or building peer-to-peer distributed agent workflows. Triggers on: A2A, agent-to-agent, AgentClient, A2AServer, agent endpoint, inter-agent communication."
---

# Agent-to-Agent (A2A) — Strands SDK

## Overview

A2A allows Strands agents to communicate over HTTP using a standard JSON protocol. One agent acts as a server (exposes an HTTP endpoint), another as a client (calls that endpoint). This enables distributed, decoupled agent systems where specialists can be deployed independently and composed at runtime.

A2A is one of three multi-agent patterns in Strands. Choose it when you need:
- Bidirectional peer collaboration between agents
- Independently deployed and versioned specialist agents
- Language- or framework-agnostic interoperability (the HTTP protocol is open)
- Service-mesh style routing and discovery

For other patterns see `strands-sdk-multiagent`.

---

## Quick Reference

| Role | Class | Purpose |
|---|---|---|
| Server | `A2AServer` | Wraps an agent, exposes HTTP endpoint |
| Client | `AgentClient` | Calls a remote agent endpoint |
| Protocol | JSON over HTTP | Standard request/response and streaming |
| Health | `GET /health` | Liveness probe for the server |

---

## A2AServer — Full Configuration

### Minimal setup

```python
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.a2a import A2AServer

agent = Agent(
    model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1"),
    system_prompt="You are a financial analysis specialist."
)

server = A2AServer(agent=agent, port=8080)
server.start()  # blocking; listens on http://0.0.0.0:8080
```

### All constructor parameters

```python
server = A2AServer(
    agent=agent,           # required — the Strands Agent instance to serve
    host="0.0.0.0",        # bind address; use "127.0.0.1" for loopback-only
    port=8080,             # TCP port
    api_key="secret-key",  # if set, enforces X-API-Key header on every request
    path="/invoke",        # URL path for the invoke endpoint (default: "/invoke")
    # Additional kwargs are passed to the underlying ASGI/uvicorn server
)
```

### Starting the server (non-blocking background thread)

```python
import threading

def run_server():
    server.start()

thread = threading.Thread(target=run_server, daemon=True)
thread.start()

# Server is now running; main thread can do other work
```

### Health check endpoint

`A2AServer` exposes `GET /health` automatically. It returns:

```json
{"status": "ok"}
```

Use this for load balancer target-group health checks, Kubernetes liveness probes, and ECS health checks.

```bash
curl http://localhost:8080/health
# {"status": "ok"}
```

---

## AgentClient — Full Configuration

### Basic usage

```python
from strands.multiagent.a2a import AgentClient

client = AgentClient(url="http://localhost:8080")
result = await client.invoke_async("Analyze Q3 revenue trends.")
print(str(result))
```

### All constructor parameters

```python
client = AgentClient(
    url="http://specialist-host:8080",   # required — base URL of the A2AServer
    api_key="secret-key",                # sent as X-API-Key header if set
    timeout=30.0,                        # request timeout in seconds (default: 30)
    max_retries=3,                       # number of retry attempts on transient errors
    headers={"X-Correlation-ID": "abc"}, # extra HTTP headers on every request
)
```

### Synchronous invocation (non-async context)

```python
import asyncio

result = asyncio.run(client.invoke_async("What is the IRR of this investment?"))
print(str(result))
```

### Wrapping a client call as a Strands tool

```python
from strands import tool
from strands.multiagent.a2a import AgentClient

@tool
async def call_financial_analyst(question: str) -> str:
    """Delegate a financial question to the specialist analyst agent.

    Args:
        question: The financial question or analysis request.

    Returns:
        Analysis result from the specialist.
    """
    client = AgentClient(url="http://finance-agent:8080", timeout=60.0)
    result = await client.invoke_async(question)
    return str(result)
```

---

## Multiple Server Agents Behind One Endpoint (Routing)

When you want a single inbound URL to dispatch to multiple specialist agents, build a thin FastAPI router in front of them.

```python
import asyncio
import threading
from fastapi import FastAPI, Request, HTTPException
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.a2a import A2AServer, AgentClient
import uvicorn

model = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1")

finance_agent  = Agent(model=model, system_prompt="You are a financial analyst.")
legal_agent    = Agent(model=model, system_prompt="You are a legal document reviewer.")
research_agent = Agent(model=model, system_prompt="You are a research summarizer.")

# Start each specialist on its own port in background threads
for agent_obj, port in [(finance_agent, 8081), (legal_agent, 8082), (research_agent, 8083)]:
    server = A2AServer(agent=agent_obj, port=port)
    threading.Thread(target=server.start, daemon=True).start()

# Router maps domain keywords to backend URLs
ROUTES = {
    "finance":   "http://localhost:8081",
    "legal":     "http://localhost:8082",
    "research":  "http://localhost:8083",
}

def classify_domain(text: str) -> str:
    text_lower = text.lower()
    if any(w in text_lower for w in ("revenue", "irr", "portfolio", "stock", "financial")):
        return "finance"
    if any(w in text_lower for w in ("contract", "clause", "liability", "legal", "compliance")):
        return "legal"
    return "research"

app = FastAPI()

@app.post("/invoke")
async def route_invoke(request: Request):
    body = await request.json()
    prompt = body.get("prompt", "")
    domain = classify_domain(prompt)
    backend_url = ROUTES[domain]
    client = AgentClient(url=backend_url, timeout=60.0)
    result = await client.invoke_async(prompt)
    return {"result": str(result), "routed_to": domain}

# Run the router on port 8080
uvicorn.run(app, host="0.0.0.0", port=8080)
```

Clients always call `http://localhost:8080/invoke` — they never need to know which specialist handled the request.

---

## A2A with Streaming Responses

Use `invoke_streaming_async` (where supported) or implement a server-sent events (SSE) wrapper around an A2AServer to forward token-level stream events to the caller.

### SSE streaming wrapper (FastAPI)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from strands import Agent
from strands.models import BedrockModel
import asyncio, json

app = FastAPI()
model = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1")
agent = Agent(model=model, system_prompt="You are a streaming research agent.")

@app.post("/invoke/stream")
async def stream_invoke(request_body: dict):
    prompt = request_body.get("prompt", "")

    async def event_generator():
        async for event in agent.stream_async(prompt):
            if event.get("type") == "content_block_delta":
                text = event.get("delta", {}).get("text", "")
                if text:
                    yield f"data: {json.dumps({'text': text})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

### Consuming the SSE stream from a client

```python
import httpx

async def stream_from_agent(prompt: str):
    async with httpx.AsyncClient(timeout=120.0) as client:
        async with client.stream("POST", "http://localhost:8080/invoke/stream",
                                  json={"prompt": prompt}) as response:
            async for line in response.aiter_lines():
                if line.startswith("data:"):
                    payload = line[5:].strip()
                    if payload == "[DONE]":
                        break
                    import json
                    chunk = json.loads(payload)
                    print(chunk["text"], end="", flush=True)
```

---

## Service Discovery — How Clients Find Server Agents

### Environment variable injection (simplest)

```python
import os
from strands.multiagent.a2a import AgentClient

FINANCE_AGENT_URL = os.environ.get("FINANCE_AGENT_URL", "http://localhost:8081")
client = AgentClient(url=FINANCE_AGENT_URL)
```

Set the variable differently per environment:

```bash
# local dev
export FINANCE_AGENT_URL=http://localhost:8081

# Docker Compose (service name resolution)
FINANCE_AGENT_URL=http://finance-agent:8080

# AWS ECS with Cloud Map
FINANCE_AGENT_URL=http://finance-agent.agents.local:8080
```

### DNS-based discovery (AWS Cloud Map / Kubernetes Service)

In Kubernetes, each A2A server is a `Service`. Clients resolve the service DNS name directly:

```python
client = AgentClient(url="http://finance-agent.agents.svc.cluster.local:8080")
```

In AWS ECS + Cloud Map, the service namespace provides DNS:

```python
client = AgentClient(url="http://finance-agent.agents.local:8080")
```

### Central registry (advanced)

```python
# Simple in-memory registry (replace with Redis or DynamoDB for production)
AGENT_REGISTRY = {
    "finance":  "http://finance-agent:8080",
    "legal":    "http://legal-agent:8080",
    "research": "http://research-agent:8080",
}

def get_agent_client(agent_name: str) -> AgentClient:
    url = AGENT_REGISTRY.get(agent_name)
    if not url:
        raise ValueError(f"Unknown agent: {agent_name}")
    return AgentClient(url=url, timeout=60.0)
```

---

## Load Balancing Multiple Server Instances

Run multiple instances of the same A2AServer behind an HTTP load balancer (AWS ALB, nginx, or HAProxy). All instances must be stateless — conversation state lives in the client prompt, not in the server.

### Docker Compose example (3 replicas)

```yaml
# docker-compose.yml
version: "3.9"

services:
  finance-agent:
    image: my-finance-agent:latest
    deploy:
      replicas: 3
    environment:
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
    ports:
      - "8081-8083:8080"   # host ports 8081-8083 map to container 8080

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
```

```nginx
# nginx.conf — round-robin upstream
upstream finance_agents {
    server finance-agent:8080;
    # Docker Compose resolves the service name to all replicas automatically
}
server {
    listen 80;
    location / {
        proxy_pass http://finance_agents;
        proxy_read_timeout 120s;
    }
}
```

Clients always call `http://localhost:8080` — the load balancer distributes requests across replicas.

---

## A2A with Authentication

### API key authentication

The simplest mechanism. Set `api_key` on both server and client.

```python
# Server
server = A2AServer(agent=agent, port=8080, api_key="change-me-in-production")

# Client
client = AgentClient(url="http://specialist:8080", api_key="change-me-in-production")
```

The server enforces the `X-API-Key` header on every request. Requests without a valid key receive `401 Unauthorized`.

Store the key in an environment variable, never hardcode it:

```python
import os
server = A2AServer(agent=agent, port=8080, api_key=os.environ["A2A_API_KEY"])
client = AgentClient(url=os.environ["SPECIALIST_URL"], api_key=os.environ["A2A_API_KEY"])
```

### JWT authentication (FastAPI middleware)

Wrap A2AServer's underlying ASGI app (or proxy it via FastAPI) to validate JWTs before forwarding requests.

```python
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt, os

SECRET = os.environ["JWT_SECRET"]
bearer_scheme = HTTPBearer()
app = FastAPI()

def verify_jwt(credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme)):
    try:
        payload = jwt.decode(credentials.credentials, SECRET, algorithms=["HS256"])
        return payload
    except jwt.PyJWTError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {e}")

@app.post("/invoke")
async def jwt_protected_invoke(request: Request, _=Depends(verify_jwt)):
    body = await request.json()
    prompt = body.get("prompt", "")
    # forward to the real A2AServer running on an internal port
    from strands.multiagent.a2a import AgentClient
    client = AgentClient(url="http://localhost:8081")  # internal, no auth
    result = await client.invoke_async(prompt)
    return {"result": str(result)}
```

Clients generate a short-lived token and pass it as `Authorization: Bearer <token>`:

```python
import jwt, time, httpx, os

def make_token() -> str:
    return jwt.encode(
        {"sub": "orchestrator", "exp": int(time.time()) + 300},
        os.environ["JWT_SECRET"],
        algorithm="HS256"
    )

async def call_specialist(prompt: str) -> str:
    token = make_token()
    async with httpx.AsyncClient() as http:
        r = await http.post(
            "http://specialist-gateway:8080/invoke",
            json={"prompt": prompt},
            headers={"Authorization": f"Bearer {token}"},
            timeout=60.0,
        )
        r.raise_for_status()
        return r.json()["result"]
```

### Mutual TLS (mTLS)

For highest security (zero-trust networks, regulated environments), enforce client certificate verification.

```python
import ssl, uvicorn
from strands.multiagent.a2a import A2AServer

server = A2AServer(agent=agent, port=8443)

ssl_ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ssl_ctx.load_cert_chain("/certs/server.crt", "/certs/server.key")
ssl_ctx.load_verify_locations("/certs/ca.crt")
ssl_ctx.verify_mode = ssl.CERT_REQUIRED  # enforce client cert

# Pass the SSL context when running uvicorn directly
uvicorn.run(server.app, host="0.0.0.0", port=8443, ssl=ssl_ctx)
```

Client side (httpx with client cert):

```python
import httpx, ssl

client_ssl = ssl.create_default_context(cafile="/certs/ca.crt")
client_ssl.load_cert_chain("/certs/client.crt", "/certs/client.key")

async with httpx.AsyncClient(verify=client_ssl) as http:
    r = await http.post("https://specialist:8443/invoke", json={"prompt": "..."})
    print(r.json()["result"])
```

---

## Async A2A Patterns

### Fire-and-forget

Submit a task to a specialist agent and return immediately without waiting for the result. Use when the result is consumed later (e.g., written to a database or queue).

```python
import asyncio
from strands.multiagent.a2a import AgentClient

async def fire_and_forget(prompt: str):
    """Submit task without waiting for result."""
    client = AgentClient(url="http://analytics-agent:8080", timeout=120.0)
    # Schedule the coroutine but do not await it
    asyncio.create_task(client.invoke_async(prompt))
    return {"status": "submitted"}
```

### Callback pattern (webhook)

The server agent calls back to a URL supplied in the request once it finishes. Useful when processing time exceeds HTTP gateway timeouts.

```python
# Orchestrator exposes a callback endpoint
from fastapi import FastAPI

app = FastAPI()
results_store: dict = {}

@app.post("/callback/{task_id}")
async def receive_callback(task_id: str, body: dict):
    results_store[task_id] = body.get("result")
    return {"ok": True}

# Orchestrator submits a task and provides its own callback URL
import httpx, uuid

async def submit_with_callback(prompt: str, my_base_url: str) -> str:
    task_id = str(uuid.uuid4())
    payload = {
        "prompt": prompt,
        "callback_url": f"{my_base_url}/callback/{task_id}",
        "task_id": task_id,
    }
    async with httpx.AsyncClient() as http:
        await http.post("http://specialist:8080/invoke/async", json=payload, timeout=10.0)
    return task_id  # caller polls results_store[task_id] or waits on an event
```

Specialist agent's async invoke endpoint:

```python
import asyncio, httpx
from fastapi import BackgroundTasks

@app.post("/invoke/async")
async def invoke_async_endpoint(body: dict, background_tasks: BackgroundTasks):
    prompt       = body["prompt"]
    callback_url = body.get("callback_url")
    task_id      = body.get("task_id", "")

    async def run_and_callback():
        from strands.multiagent.a2a import AgentClient
        client = AgentClient(url="http://localhost:8081")  # internal agent
        result = await client.invoke_async(prompt)
        if callback_url:
            async with httpx.AsyncClient() as http:
                await http.post(callback_url, json={"task_id": task_id, "result": str(result)})

    background_tasks.add_task(run_and_callback)
    return {"task_id": task_id, "status": "accepted"}
```

### Parallel fan-out to multiple specialists

```python
import asyncio
from strands.multiagent.a2a import AgentClient

async def parallel_specialists(prompt: str) -> dict:
    clients = {
        "finance":  AgentClient(url="http://finance-agent:8080",  timeout=60.0),
        "legal":    AgentClient(url="http://legal-agent:8080",    timeout=60.0),
        "research": AgentClient(url="http://research-agent:8080", timeout=60.0),
    }
    tasks = {name: client.invoke_async(prompt) for name, client in clients.items()}
    results = await asyncio.gather(*tasks.values(), return_exceptions=True)
    return {
        name: str(r) if not isinstance(r, Exception) else f"ERROR: {r}"
        for name, r in zip(tasks.keys(), results)
    }
```

---

## Error Handling and Circuit Breaker Pattern

### Basic error handling around AgentClient

```python
import logging
from strands.multiagent.a2a import AgentClient
import httpx

logger = logging.getLogger(__name__)

async def safe_a2a_call(url: str, prompt: str, fallback: str = "") -> str:
    client = AgentClient(url=url, timeout=30.0, max_retries=2)
    try:
        result = await client.invoke_async(prompt)
        return str(result)
    except httpx.TimeoutException:
        logger.warning(f"A2A call to {url} timed out")
        return fallback
    except httpx.HTTPStatusError as e:
        logger.error(f"A2A server returned {e.response.status_code}: {url}")
        return fallback
    except Exception as e:
        logger.error(f"Unexpected A2A error for {url}: {e}", exc_info=True)
        return fallback
```

### Circuit breaker

Prevents cascading failures when a downstream agent is unhealthy. After `failure_threshold` consecutive failures the circuit opens and fast-fails for `recovery_timeout` seconds before trying again.

```python
import asyncio, time, logging
from enum import Enum
from strands.multiagent.a2a import AgentClient

logger = logging.getLogger(__name__)

class CircuitState(Enum):
    CLOSED   = "closed"    # normal operation
    OPEN     = "open"      # failing fast
    HALF_OPEN = "half_open" # testing recovery

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 3, recovery_timeout: float = 30.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout  = recovery_timeout
        self.failures          = 0
        self.state             = CircuitState.CLOSED
        self.last_failure_time = 0.0

    def record_success(self):
        self.failures = 0
        self.state    = CircuitState.CLOSED

    def record_failure(self):
        self.failures += 1
        self.last_failure_time = time.monotonic()
        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN
            logger.warning("Circuit breaker OPENED")

    def allow_request(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        if self.state == CircuitState.OPEN:
            if time.monotonic() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False
        return True  # HALF_OPEN allows one probe

# Global circuit breakers per downstream agent
_breakers: dict[str, CircuitBreaker] = {}

def get_breaker(url: str) -> CircuitBreaker:
    if url not in _breakers:
        _breakers[url] = CircuitBreaker()
    return _breakers[url]

async def guarded_a2a_call(url: str, prompt: str) -> str:
    breaker = get_breaker(url)
    if not breaker.allow_request():
        raise RuntimeError(f"Circuit breaker OPEN for {url} — downstream agent unavailable")

    client = AgentClient(url=url, timeout=30.0)
    try:
        result = await client.invoke_async(prompt)
        breaker.record_success()
        return str(result)
    except Exception as e:
        breaker.record_failure()
        raise
```

### Retry with exponential backoff (A2A-specific)

```python
import asyncio
from strands.multiagent.a2a import AgentClient
import httpx

async def invoke_with_backoff(url: str, prompt: str,
                               max_attempts: int = 3,
                               base_delay: float = 1.0) -> str:
    client = AgentClient(url=url, timeout=30.0)
    for attempt in range(max_attempts):
        try:
            result = await client.invoke_async(prompt)
            return str(result)
        except (httpx.TimeoutException, httpx.ConnectError) as e:
            if attempt == max_attempts - 1:
                raise
            delay = base_delay * (2 ** attempt)
            await asyncio.sleep(delay)
    raise RuntimeError("Unreachable")
```

---

## Deploying A2A Servers

### Docker

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY agent_server.py .

EXPOSE 8080

CMD ["python", "agent_server.py"]
```

```python
# agent_server.py
import os
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.a2a import A2AServer

model = BedrockModel(
    model_id=os.environ.get("BEDROCK_MODEL_ID", "anthropic.claude-haiku-4-5-20251001-v1:0"),
    region_name=os.environ.get("AWS_REGION", "us-east-1"),
)

agent = Agent(model=model, system_prompt=os.environ.get("AGENT_SYSTEM_PROMPT", "You are a helpful assistant."))

server = A2AServer(
    agent=agent,
    host="0.0.0.0",
    port=int(os.environ.get("PORT", "8080")),
    api_key=os.environ.get("A2A_API_KEY"),
)
server.start()
```

```bash
docker build -t finance-agent .
docker run -p 8080:8080 \
  -e AWS_REGION=us-east-1 \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  -e AGENT_SYSTEM_PROMPT="You are a financial analyst." \
  -e A2A_API_KEY=change-me \
  finance-agent
```

### AWS ECS (Fargate)

Key ECS task definition fields:

```json
{
  "family": "finance-agent",
  "containerDefinitions": [{
    "name": "finance-agent",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/finance-agent:latest",
    "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
    "environment": [
      {"name": "AWS_REGION", "value": "us-east-1"},
      {"name": "BEDROCK_MODEL_ID", "value": "anthropic.claude-haiku-4-5-20251001-v1:0"}
    ],
    "secrets": [
      {"name": "A2A_API_KEY", "valueFrom": "arn:aws:secretsmanager:us-east-1:...:secret:a2a-api-key"}
    ],
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
      "interval": 15,
      "timeout": 5,
      "retries": 3
    }
  }],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "taskRoleArn": "arn:aws:iam::...:role/AgentTaskRole"
}
```

The `taskRoleArn` must grant `bedrock:InvokeModel` on the model ARN — no access keys needed inside the container when using task roles.

### AWS Lambda (short-lived, synchronous only)

Lambda is appropriate for lightweight agents with sub-15-minute response times. Use Mangum to wrap the FastAPI app.

```python
# lambda_handler.py
from mangum import Mangum
from fastapi import FastAPI
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.a2a import A2AServer
import os

model = BedrockModel(
    model_id=os.environ["BEDROCK_MODEL_ID"],
    region_name=os.environ["AWS_REGION"],
)
agent = Agent(model=model, system_prompt=os.environ.get("SYSTEM_PROMPT", ""))

# Reuse the A2AServer's underlying ASGI app
a2a_server = A2AServer(agent=agent, port=8080)
# Expose the ASGI app to Mangum
handler = Mangum(a2a_server.app)
```

Lambda environment variable `BEDROCK_MODEL_ID` and a resource-based policy or execution role granting `bedrock:InvokeModel` are required.

---

## Testing A2A Setups Locally

### Run server and client in the same process (integration test)

```python
import asyncio, threading, time
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.a2a import A2AServer, AgentClient

def start_test_server():
    model  = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1")
    agent  = Agent(model=model, system_prompt="Reply with 'pong' to any message.")
    server = A2AServer(agent=agent, host="127.0.0.1", port=19999)
    server.start()

# Start server in background thread
thread = threading.Thread(target=start_test_server, daemon=True)
thread.start()
time.sleep(1.0)  # wait for server to bind

async def test_round_trip():
    client = AgentClient(url="http://127.0.0.1:19999", timeout=15.0)
    result = await client.invoke_async("ping")
    assert "pong" in str(result).lower(), f"Unexpected response: {result}"
    print("Round-trip test passed.")

asyncio.run(test_round_trip())
```

### Health check probe in tests

```python
import httpx

async def wait_for_server(url: str, timeout: float = 10.0):
    import time
    deadline = time.monotonic() + timeout
    async with httpx.AsyncClient() as http:
        while time.monotonic() < deadline:
            try:
                r = await http.get(f"{url}/health")
                if r.status_code == 200:
                    return
            except httpx.ConnectError:
                pass
            await asyncio.sleep(0.2)
    raise TimeoutError(f"Server at {url} did not become healthy within {timeout}s")
```

### Mocking AgentClient for unit tests

When unit-testing an orchestrator that calls specialists, mock `AgentClient.invoke_async` to avoid real HTTP calls.

```python
from unittest.mock import AsyncMock, patch

async def test_orchestrator_delegates():
    with patch("strands.multiagent.a2a.AgentClient.invoke_async",
               new_callable=AsyncMock) as mock_invoke:
        mock_invoke.return_value = "Mocked specialist response"
        result = await call_financial_analyst("What is the P/E ratio?")
        assert result == "Mocked specialist response"
        mock_invoke.assert_called_once_with("What is the P/E ratio?")
```

### pytest fixture for a temporary A2AServer

```python
import pytest, threading, time
from strands import Agent
from strands.models import BedrockModel
from strands.multiagent.a2a import A2AServer

@pytest.fixture(scope="session")
def test_server():
    model  = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0", region_name="us-east-1")
    agent  = Agent(model=model, system_prompt="You are a test echo agent.")
    server = A2AServer(agent=agent, host="127.0.0.1", port=19998)
    thread = threading.Thread(target=server.start, daemon=True)
    thread.start()
    time.sleep(0.5)
    yield "http://127.0.0.1:19998"
    # daemon thread exits automatically when the test session ends
```

---

## Expanded Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `ConnectionRefusedError` on client call | Server not started or wrong port | Check `server.start()` was called; verify port matches in both server and client |
| `401 Unauthorized` from server | Missing or wrong `api_key` | Ensure `api_key` is identical on server and client; check for trailing whitespace in env var |
| `httpx.TimeoutException` | Agent taking longer than `timeout` | Increase `timeout=` on `AgentClient`; check server logs for slow LLM calls |
| Server starts but `POST /invoke` returns 404 | Custom `path=` mismatch | Ensure `AgentClient(url=...)` base URL does not include the path, or that both sides agree on `path=` |
| `RuntimeError: no running event loop` | `invoke_async` called in sync context | Wrap in `asyncio.run(client.invoke_async(...))` |
| Circuit breaker opens immediately | `failure_threshold=1` or server crashing on every request | Check server logs; increase threshold; fix underlying agent error first |
| High latency on first request | Cold-start model initialization | Pre-warm agent by invoking a no-op prompt on server startup |
| `AccessDeniedException` (Bedrock) inside server container | Task role or IAM user missing `bedrock:InvokeModel` | Add policy; for ECS use task role, not access keys in env |
| `ValidationException: model identifier` | Wrong `BEDROCK_MODEL_ID` or wrong region | Verify ID in Bedrock console for the target region |
| Agent returns empty string | `str(result)` on an object | Use `str(result).strip()` or `result.message` |
| Docker container exits immediately | Unhandled exception in `agent_server.py` | Check container logs: `docker logs <id>`; ensure env vars are set |
| Load balancer health check fails | `/health` path not reachable | Confirm security group allows health check port; check ALB target group path is `/health` |
| mTLS handshake failure | Certificate chain mismatch | Verify CA cert is the issuer of both server and client certs; check cert expiry |
| Streaming client gets no data | `text/event-stream` headers stripped | Check nginx/ALB is not buffering SSE; set `proxy_buffering off` in nginx |
| `max_retries` not retrying | Exception type not in retry list | `AgentClient` retries on transient network errors; catch and re-raise at a higher level for other errors |
| JWT `401` on every request | Clock skew between issuer and verifier | Synchronize system clocks; add `leeway=5` to `jwt.decode()` |

### Debug logging for A2A

```python
import logging

# Show all HTTP traffic between client and server
logging.getLogger("httpx").setLevel(logging.DEBUG)
logging.getLogger("strands").setLevel(logging.DEBUG)
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s %(name)s %(levelname)s %(message)s"
)
```

### Verify server is running

```bash
# Health check
curl http://localhost:8080/health

# Manual invoke (no auth)
curl -X POST http://localhost:8080/invoke \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Say hello."}'

# Manual invoke (with API key)
curl -X POST http://localhost:8080/invoke \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-secret-key" \
  -d '{"prompt": "Say hello."}'
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Server not started before client call | Ensure `server.start()` (or background thread) runs and binds before the first `invoke_async` |
| Port conflict | Change port or `lsof -i :8080` to find the blocking process |
| Firewall blocks inter-agent calls | Open the port in security group / firewall rules |
| `invoke_async` called in sync context | Use `asyncio.run(client.invoke_async(...))` |
| Hardcoded `api_key` in source | Read from `os.environ`; never commit secrets |
| Creating a new `AgentClient` per request | Create once and reuse; `AgentClient` manages its own connection pool |
| Forgetting `await` on `invoke_async` | Missing `await` returns a coroutine object, not a string |
| No timeout set on client | Default may be too short or too long; always set `timeout=` explicitly |
| Using SSO role credentials in container | Use ECS task role with `bedrock:InvokeModel`; SSO roles often carry explicit Deny policies |
