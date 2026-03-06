---
name: strands-sdk-streaming
description: "Use when streaming agent responses token-by-token, handling StreamEvent callbacks, printing output as it arrives, integrating Strands streaming into a WebSocket or HTTP SSE endpoint, or using async generator response iteration. Triggers on: stream, stream_async, StreamEvent, callback_handler, yield tokens, real-time output, SSE."
---

# Streaming — Strands SDK

## Overview

Strands supports streaming responses so you can process tokens as they arrive instead of waiting for the full response. Use `stream_async` for async contexts or `callback_handler` for sync streaming. Streaming is essential for chat UIs, voice pipelines, SSE endpoints, and any application where perceived latency matters.

---

## Quick Reference

| Method | Context | Use case |
|---|---|---|
| `agent.stream_async(prompt)` | async | Async generator — yields StreamEvents as they arrive |
| `callback_handler=fn` | sync or async | Invoked per token/event; simpler integration |
| Event type `content_block_start` | — | Signals start of a new content block |
| Event type `content_block_delta` | — | Contains actual text tokens (`text_delta`) or tool input fragments (`input_json_delta`) |
| Event type `content_block_stop` | — | Signals end of a content block |
| Event type `message_start` | — | First event; carries initial usage estimates |
| Event type `message_delta` | — | Contains `stop_reason` and final usage counts |
| Event type `message_stop` | — | Terminal event; generation is complete |
| Event type `tool_use` | — | Emitted when the model decides to call a tool |
| Event type `metadata` | — | Carries latency, request ID, and other provider metadata |

---

## Full Event Type Reference

Strands streaming surfaces the Anthropic/Bedrock streaming event protocol. Each event is a plain Python `dict`.

### `message_start`

Emitted once at the beginning of the stream. Contains initial token usage (input tokens are known at this point).

```python
{
    "type": "message_start",
    "message": {
        "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
        "type": "message",
        "role": "assistant",
        "content": [],
        "model": "anthropic.claude-haiku-4-5-20251001-v1:0",
        "stop_reason": None,
        "stop_sequence": None,
        "usage": {
            "input_tokens": 25,
            "output_tokens": 1
        }
    }
}
```

### `content_block_start`

Signals that a new content block is beginning. `index` is zero-based. Text blocks have `type: "text"`. Tool use blocks have `type: "tool_use"` and include the tool `id` and `name`.

```python
# Text block
{
    "type": "content_block_start",
    "index": 0,
    "content_block": {"type": "text", "text": ""}
}

# Tool use block
{
    "type": "content_block_start",
    "index": 1,
    "content_block": {
        "type": "tool_use",
        "id": "toolu_01A09q90qw90lq917835lq9",
        "name": "get_weather",
        "input": {}
    }
}
```

### `content_block_delta`

The main event carrying actual content. Emitted repeatedly until the block is complete. Two delta subtypes exist:

```python
# Text token — most common case
{
    "type": "content_block_delta",
    "index": 0,
    "delta": {
        "type": "text_delta",
        "text": "Neural networks are"
    }
}

# Tool input fragment — JSON streamed incrementally
{
    "type": "content_block_delta",
    "index": 1,
    "delta": {
        "type": "input_json_delta",
        "partial_json": "{\"city\": \"Par"
    }
}
```

### `content_block_stop`

Signals that block at `index` is finished. After this, the block's content is complete.

```python
{"type": "content_block_stop", "index": 0}
```

### `message_delta`

Emitted near the end. Carries `stop_reason` (`"end_turn"`, `"tool_use"`, `"max_tokens"`, `"stop_sequence"`) and final cumulative usage.

```python
{
    "type": "message_delta",
    "delta": {
        "stop_reason": "end_turn",
        "stop_sequence": None
    },
    "usage": {
        "output_tokens": 147
    }
}
```

### `message_stop`

Final event. Signals the stream is fully exhausted. No payload beyond the `type` key.

```python
{"type": "message_stop"}
```

### `metadata`

Provider-level metadata. Timing and request tracing. May appear at stream start or end depending on the provider.

```python
{
    "type": "metadata",
    "request_id": "req_01EXaMPLe",
    "latency_ms": 312
}
```

---

## Async Streaming

The primary pattern for async applications (FastAPI, asyncio scripts, WebSocket handlers).

```python
import asyncio
from strands import Agent
from strands.models import BedrockModel

agent = Agent(model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                                  region_name="us-east-1"))

async def main():
    async for event in agent.stream_async("Explain neural networks."):
        event_type = event.get("type")

        if event_type == "content_block_delta":
            delta = event.get("delta", {})
            if delta.get("type") == "text_delta":
                print(delta.get("text", ""), end="", flush=True)

        elif event_type == "message_delta":
            stop_reason = event.get("delta", {}).get("stop_reason")
            output_tokens = event.get("usage", {}).get("output_tokens", 0)
            print(f"\n[stop={stop_reason}, output_tokens={output_tokens}]")

        elif event_type == "message_stop":
            print("\n[stream complete]")

asyncio.run(main())
```

---

## Sync Streaming via Callback

For synchronous contexts or when you prefer a callback style instead of an async generator.

```python
from strands import Agent
from strands.models import BedrockModel

tokens = []

def on_event(event):
    if event.get("type") == "content_block_delta":
        text = event.get("delta", {}).get("text", "")
        if text:
            tokens.append(text)
            print(text, end="", flush=True)
    elif event.get("type") == "message_stop":
        print()  # newline at end

agent = Agent(
    model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                       region_name="us-east-1"),
    callback_handler=on_event
)
agent("Write a poem about the ocean.")
full_text = "".join(tokens)
```

---

## Streaming with Tool Calls

When the model decides to invoke a tool, the stream emits a `tool_use` content block. Text and tool-use blocks can be interleaved. To handle both:

```python
import json
import asyncio
from strands import Agent, tool
from strands.models import BedrockModel

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"Sunny, 22C in {city}"

agent = Agent(
    model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                       region_name="us-east-1"),
    tools=[get_weather]
)

async def stream_with_tools(prompt: str):
    # Track per-block-index state for assembling tool input JSON
    block_types = {}        # index -> "text" | "tool_use"
    tool_names = {}         # index -> tool name
    tool_input_buffers = {} # index -> partial JSON string
    text_buffer = []

    async for event in agent.stream_async(prompt):
        etype = event.get("type")

        if etype == "content_block_start":
            idx = event["index"]
            cb = event.get("content_block", {})
            block_types[idx] = cb.get("type")
            if cb.get("type") == "tool_use":
                tool_names[idx] = cb.get("name")
                tool_input_buffers[idx] = ""
                print(f"\n[tool call starting: {cb.get('name')}]")

        elif etype == "content_block_delta":
            idx = event["index"]
            delta = event.get("delta", {})
            dtype = delta.get("type")

            if dtype == "text_delta":
                text = delta.get("text", "")
                text_buffer.append(text)
                print(text, end="", flush=True)

            elif dtype == "input_json_delta":
                # Accumulate partial JSON for tool input
                tool_input_buffers[idx] = tool_input_buffers.get(idx, "") + delta.get("partial_json", "")

        elif etype == "content_block_stop":
            idx = event["index"]
            if block_types.get(idx) == "tool_use":
                raw = tool_input_buffers.get(idx, "{}")
                try:
                    tool_input = json.loads(raw)
                except json.JSONDecodeError:
                    tool_input = {}
                print(f"\n[tool input assembled: {tool_input}]")
                # Strands executes tools automatically; this is for observability only

        elif etype == "message_delta":
            stop_reason = event.get("delta", {}).get("stop_reason")
            print(f"\n[stop_reason={stop_reason}]")

asyncio.run(stream_with_tools("What is the weather in Paris?"))
```

Key points:
- Tool use blocks stream their `input` as `input_json_delta` fragments — accumulate them before `json.loads`.
- `stop_reason == "tool_use"` in `message_delta` means the model paused to call a tool; Strands handles execution and re-invokes the model automatically.
- The final text after tool results has `stop_reason == "end_turn"`.

---

## Buffering Strategies

### Strategy 1: Process per token (lowest latency)

Best for streaming UIs, voice pipelines, and live displays. Minimal memory footprint. Each token is forwarded immediately.

```python
async def stream_per_token(agent, prompt, send_fn):
    """Forward tokens as they arrive. send_fn is any async callable."""
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                await send_fn(text)
    await send_fn(None)  # sentinel: stream complete
```

### Strategy 2: Accumulate full response (simplest, highest latency)

Use when you need the complete text before acting (e.g., parsing structured JSON output, full-response caching).

```python
async def stream_accumulate(agent, prompt) -> str:
    """Collect all tokens and return the complete response string."""
    parts = []
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                parts.append(text)
    return "".join(parts)
```

### Strategy 3: Sentence-boundary buffering (TTS / chunked delivery)

Buffer tokens until a sentence boundary, then flush. Balances latency with chunk coherence — useful for TTS pipelines where sub-sentence chunks sound unnatural.

```python
import re

SENTENCE_END = re.compile(r'(?<=[.!?])\s+')

async def stream_by_sentence(agent, prompt, flush_fn):
    """
    Accumulate tokens and flush on sentence boundaries.
    flush_fn receives complete sentence strings.
    """
    buffer = ""
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if not text:
                continue
            buffer += text
            # Check for sentence boundary in buffer
            parts = SENTENCE_END.split(buffer)
            if len(parts) > 1:
                # All complete sentences except the last (potentially incomplete) fragment
                for sentence in parts[:-1]:
                    sentence = sentence.strip()
                    if sentence:
                        await flush_fn(sentence)
                buffer = parts[-1]  # keep remainder

        elif event.get("type") == "message_stop":
            # Flush any remaining text
            remainder = buffer.strip()
            if remainder:
                await flush_fn(remainder)
            await flush_fn(None)  # sentinel
```

### Strategy 4: Word-boundary buffering

Flush every N words. Useful for streaming to translation APIs or chunked logging.

```python
async def stream_by_words(agent, prompt, flush_fn, chunk_size=10):
    word_buffer = []
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                word_buffer.extend(text.split(" "))
                while len(word_buffer) >= chunk_size:
                    chunk = " ".join(word_buffer[:chunk_size])
                    await flush_fn(chunk)
                    word_buffer = word_buffer[chunk_size:]
    if word_buffer:
        await flush_fn(" ".join(word_buffer))
    await flush_fn(None)
```

---

## FastAPI SSE Endpoint (Complete Working Example)

Server-Sent Events deliver a server-push stream over a plain HTTP connection. The browser `EventSource` API consumes them natively with no WebSocket setup required.

```python
import asyncio
import json
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from strands import Agent
from strands.models import BedrockModel

app = FastAPI()

model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1"
)

# Create once at startup; reuse across requests
agent = Agent(model=model, system_prompt="You are a helpful assistant.")


def sse_event(data: dict, event: str = "message") -> str:
    """Format a dict as an SSE frame."""
    return f"event: {event}\ndata: {json.dumps(data)}\n\n"


async def token_generator(prompt: str):
    """Async generator that yields SSE-formatted frames."""
    try:
        async for event in agent.stream_async(prompt):
            etype = event.get("type")

            if etype == "content_block_delta":
                text = event.get("delta", {}).get("text", "")
                if text:
                    yield sse_event({"text": text}, event="token")

            elif etype == "message_delta":
                usage = event.get("usage", {})
                stop = event.get("delta", {}).get("stop_reason")
                yield sse_event(
                    {"stop_reason": stop, "output_tokens": usage.get("output_tokens", 0)},
                    event="usage"
                )

            elif etype == "message_stop":
                yield sse_event({"status": "done"}, event="done")
                return

    except Exception as e:
        yield sse_event({"error": str(e)}, event="error")


@app.get("/stream")
async def stream_endpoint(prompt: str):
    return StreamingResponse(
        token_generator(prompt),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",   # disable nginx buffering
            "Connection": "keep-alive",
        }
    )
```

**Browser client:**

```javascript
const source = new EventSource(`/stream?prompt=Explain+quantum+computing`);
let fullText = "";

source.addEventListener("token", (e) => {
    const { text } = JSON.parse(e.data);
    fullText += text;
    document.getElementById("output").textContent = fullText;
});

source.addEventListener("done", () => {
    source.close();
});

source.addEventListener("error", (e) => {
    console.error("Stream error", e);
    source.close();
});
```

---

## WebSocket Broadcast to Multiple Clients

Broadcast each token to all connected clients simultaneously using an async queue fan-out pattern.

```python
import asyncio
import json
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from strands import Agent
from strands.models import BedrockModel

app = FastAPI()
model = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                     region_name="us-east-1")
agent = Agent(model=model)

# Registry of active client queues
_client_queues: list[asyncio.Queue] = []


async def broadcast_stream(prompt: str):
    """Run the stream and push each token to every registered client queue."""
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                msg = json.dumps({"type": "token", "text": text})
                for q in list(_client_queues):
                    await q.put(msg)
        elif event.get("type") == "message_stop":
            done_msg = json.dumps({"type": "done"})
            for q in list(_client_queues):
                await q.put(done_msg)
                await q.put(None)  # sentinel to close each client loop


@app.websocket("/ws/broadcast")
async def ws_broadcast(websocket: WebSocket):
    await websocket.accept()

    queue: asyncio.Queue = asyncio.Queue(maxsize=256)
    _client_queues.append(queue)

    try:
        while True:
            item = await queue.get()
            if item is None:
                break
            await websocket.send_text(item)
    except WebSocketDisconnect:
        pass
    finally:
        _client_queues.remove(queue)


@app.post("/broadcast")
async def start_broadcast(prompt: str):
    """Trigger a broadcast stream to all connected WebSocket clients."""
    asyncio.create_task(broadcast_stream(prompt))
    return {"status": "streaming", "clients": len(_client_queues)}
```

For individual per-connection streaming (each client gets their own independent stream):

```python
@app.websocket("/ws/stream")
async def ws_per_client(websocket: WebSocket):
    await websocket.accept()
    try:
        data = await websocket.receive_text()
        payload = json.loads(data)
        prompt = payload.get("prompt", "")

        async for event in agent.stream_async(prompt):
            if event.get("type") == "content_block_delta":
                text = event.get("delta", {}).get("text", "")
                if text:
                    await websocket.send_json({"type": "token", "text": text})
            elif event.get("type") == "message_stop":
                await websocket.send_json({"type": "done"})

    except WebSocketDisconnect:
        pass
```

---

## Streaming with Error Recovery (Reconnect on Disconnect)

For long-running streams over unreliable connections, implement client-side reconnect with a `Last-Event-ID` cursor so the stream can resume mid-response without starting over.

**Server (cursor-aware SSE):**

```python
import asyncio
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from strands import Agent
from strands.models import BedrockModel

app = FastAPI()
agent = Agent(model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                                  region_name="us-east-1"))


async def resumable_token_generator(prompt: str, start_index: int = 0):
    """
    Yield SSE frames with an id field so clients can reconnect mid-stream.
    start_index: skip tokens already delivered (from Last-Event-ID header).
    """
    token_index = 0
    try:
        async for event in agent.stream_async(prompt):
            if event.get("type") == "content_block_delta":
                text = event.get("delta", {}).get("text", "")
                if text:
                    if token_index >= start_index:
                        yield f"id: {token_index}\nevent: token\ndata: {json.dumps({'text': text})}\n\n"
                    token_index += 1

            elif event.get("type") == "message_stop":
                yield f"event: done\ndata: {{}}\n\n"
                return

    except asyncio.CancelledError:
        # Client disconnected — clean up gracefully
        return
    except Exception as e:
        yield f"event: error\ndata: {json.dumps({'error': str(e)})}\n\n"


@app.get("/stream/resumable")
async def resumable_stream(request: Request, prompt: str):
    last_event_id = request.headers.get("Last-Event-ID", "0")
    start_index = int(last_event_id) + 1 if last_event_id != "0" else 0

    return StreamingResponse(
        resumable_token_generator(prompt, start_index),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"}
    )
```

**Browser client with auto-reconnect:**

```javascript
function connectStream(prompt) {
    const url = `/stream/resumable?prompt=${encodeURIComponent(prompt)}`;
    const source = new EventSource(url);

    source.addEventListener("token", (e) => {
        document.getElementById("out").textContent += JSON.parse(e.data).text;
    });

    source.addEventListener("done", () => source.close());

    // EventSource reconnects automatically using Last-Event-ID from the `id:` field.
    // The browser sends the last received id on reconnect — no extra code needed.
    source.onerror = (e) => {
        if (source.readyState === EventSource.CLOSED) {
            console.log("Stream closed.");
        }
        // Browser will automatically retry after 3 seconds (default SSE retry interval)
    };
}
```

**Python client with manual retry (non-browser):**

```python
import asyncio
import httpx

async def stream_with_retry(url: str, prompt: str, max_retries: int = 5):
    last_id = 0
    attempt = 0

    while attempt < max_retries:
        try:
            headers = {"Last-Event-ID": str(last_id)} if last_id else {}
            async with httpx.AsyncClient(timeout=None) as client:
                async with client.stream("GET", url,
                                         params={"prompt": prompt},
                                         headers=headers) as response:
                    async for line in response.aiter_lines():
                        if line.startswith("id:"):
                            last_id = int(line[3:].strip())
                        elif line.startswith("data:") and "done" not in line:
                            import json
                            data = json.loads(line[5:])
                            print(data.get("text", ""), end="", flush=True)
                        elif "done" in line:
                            return
            return  # clean completion

        except (httpx.ReadError, httpx.ConnectError) as e:
            attempt += 1
            wait = 2 ** attempt
            print(f"\nDisconnected (attempt {attempt}), retrying in {wait}s: {e}")
            await asyncio.sleep(wait)

    raise RuntimeError(f"Stream failed after {max_retries} retries")
```

---

## Progress Tracking During Streaming

Track how far through a generation you are using token counts and elapsed time. Useful for progress bars, timeout guards, and cost estimation.

```python
import asyncio
import time
from dataclasses import dataclass, field

from strands import Agent
from strands.models import BedrockModel


@dataclass
class StreamProgress:
    input_tokens: int = 0
    output_tokens: int = 0
    output_tokens_estimate: int = 0  # from message_start (Bedrock estimate)
    chars_received: int = 0
    started_at: float = field(default_factory=time.time)
    first_token_at: float | None = None
    completed: bool = False

    @property
    def elapsed_seconds(self) -> float:
        return time.time() - self.started_at

    @property
    def time_to_first_token_ms(self) -> float | None:
        if self.first_token_at is None:
            return None
        return (self.first_token_at - self.started_at) * 1000

    @property
    def tokens_per_second(self) -> float:
        elapsed = self.elapsed_seconds
        return self.output_tokens / elapsed if elapsed > 0 else 0.0

    def percent_complete(self) -> float | None:
        """Estimate % complete based on token counts (rough estimate only)."""
        if self.output_tokens_estimate > 0:
            return min(100.0, (self.output_tokens / self.output_tokens_estimate) * 100)
        return None


async def stream_with_progress(agent, prompt: str, on_token=None, on_progress=None):
    progress = StreamProgress()
    text_parts = []

    async for event in agent.stream_async(prompt):
        etype = event.get("type")

        if etype == "message_start":
            usage = event.get("message", {}).get("usage", {})
            progress.input_tokens = usage.get("input_tokens", 0)
            # Bedrock sometimes includes a predicted output_tokens estimate here
            progress.output_tokens_estimate = usage.get("output_tokens", 0)

        elif etype == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                if progress.first_token_at is None:
                    progress.first_token_at = time.time()
                progress.chars_received += len(text)
                text_parts.append(text)
                if on_token:
                    await on_token(text)
                if on_progress:
                    await on_progress(progress)

        elif etype == "message_delta":
            output_tokens = event.get("usage", {}).get("output_tokens", 0)
            progress.output_tokens = output_tokens

        elif etype == "message_stop":
            progress.completed = True
            if on_progress:
                await on_progress(progress)

    return "".join(text_parts), progress


# Usage example
async def main():
    agent = Agent(model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                                     region_name="us-east-1"))

    async def show_progress(p: StreamProgress):
        pct = p.percent_complete()
        ttft = p.time_to_first_token_ms
        print(
            f"\r[tokens={p.output_tokens} | {p.tokens_per_second:.1f} tok/s"
            f" | ttft={ttft:.0f}ms | {pct:.0f}% est.]" if pct and ttft else
            f"\r[tokens={p.output_tokens} | {p.tokens_per_second:.1f} tok/s]",
            end="", flush=True
        )

    text, stats = await stream_with_progress(
        agent,
        "Explain transformer architecture in detail.",
        on_token=lambda t: asyncio.coroutine(lambda: print(t, end="", flush=True))(),
        on_progress=show_progress
    )
    print(f"\nDone: {stats.output_tokens} tokens in {stats.elapsed_seconds:.2f}s "
          f"(TTFT={stats.time_to_first_token_ms:.0f}ms)")
```

---

## Streaming to Audio TTS Pipelines

The token-to-TTS-queue pattern: stream LLM tokens, buffer until a natural speech boundary (sentence or clause), then enqueue each chunk to a TTS worker. This minimizes the delay between LLM start and the first audio playing.

```python
import asyncio
import re
from strands import Agent
from strands.models import BedrockModel

# Sentence boundary: period/exclamation/question followed by whitespace
SENTENCE_RE = re.compile(r'(?<=[.!?])\s+')
# Also split on em-dash or semicolon for shorter, more natural TTS chunks
CLAUSE_RE = re.compile(r'(?<=[;])\s+|(?<=—)')


async def tts_worker(tts_queue: asyncio.Queue, tts_client, websocket):
    """
    Consume sentence chunks from the queue, synthesize audio, and send over WebSocket.
    Runs as a concurrent asyncio Task — does not block the streaming loop.
    """
    import base64
    while True:
        sentence = await tts_queue.get()
        if sentence is None:
            tts_queue.task_done()
            break
        try:
            audio_bytes = await tts_client.synthesize(sentence)
            if audio_bytes:
                await websocket.send_json({
                    "type": "audio_chunk",
                    "audio_data": base64.b64encode(audio_bytes).decode(),
                    "text": sentence,
                })
        except Exception as e:
            # TTS failure is non-fatal — text was already sent
            import logging
            logging.getLogger(__name__).error("TTS error: %s", e)
        finally:
            tts_queue.task_done()


async def stream_to_tts(agent, prompt: str, tts_client, websocket):
    """
    Stream LLM response, buffer by sentence, and feed each sentence to TTS concurrently.
    Text tokens are also sent immediately to the frontend for display.
    """
    tts_queue: asyncio.Queue = asyncio.Queue(maxsize=16)

    # Start TTS worker as a concurrent task
    worker_task = asyncio.create_task(tts_worker(tts_queue, tts_client, websocket))

    text_buffer = ""
    async for event in agent.stream_async(prompt):
        etype = event.get("type")

        if etype == "content_block_delta":
            token = event.get("delta", {}).get("text", "")
            if not token:
                continue

            # Forward token immediately for live text display
            await websocket.send_json({"type": "token", "text": token})

            text_buffer += token

            # Look for sentence boundaries in the accumulated buffer
            sentences = SENTENCE_RE.split(text_buffer)
            if len(sentences) > 1:
                for sentence in sentences[:-1]:
                    sentence = sentence.strip()
                    if sentence:
                        await tts_queue.put(sentence)
                text_buffer = sentences[-1]

        elif etype == "message_stop":
            # Flush remaining buffer
            remainder = text_buffer.strip()
            if remainder:
                await tts_queue.put(remainder)
            await tts_queue.put(None)  # shutdown sentinel
            break

    # Wait for all TTS chunks to be processed
    await tts_queue.join()
    await worker_task

    await websocket.send_json({"type": "done"})
```

Key design decisions in this pattern:
- The TTS worker runs as a **concurrent Task** so TTS synthesis does not stall the streaming loop.
- The queue has `maxsize=16` to apply backpressure when TTS falls behind the LLM.
- Text tokens are forwarded immediately for text display, independently of TTS completion.
- `tts_queue.join()` ensures all audio has been sent before the `done` message fires.

---

## Performance Optimization

### Backpressure: Slow Consumer Handling

When a consumer (e.g., WebSocket client, TTS service) is slower than the LLM produces tokens, an unbounded buffer will grow until memory exhaustion. Apply backpressure with a bounded queue.

```python
import asyncio

async def stream_with_backpressure(agent, prompt: str, send_fn, maxsize: int = 32):
    """
    Buffer tokens in a bounded queue. If the consumer is slow, the producer
    awaits queue space — applying natural backpressure to the async scheduler.
    """
    queue: asyncio.Queue = asyncio.Queue(maxsize=maxsize)

    async def producer():
        async for event in agent.stream_async(prompt):
            if event.get("type") == "content_block_delta":
                text = event.get("delta", {}).get("text", "")
                if text:
                    await queue.put(text)      # blocks if queue full
        await queue.put(None)                  # sentinel

    async def consumer():
        while True:
            token = await queue.get()
            if token is None:
                break
            await send_fn(token)               # slow consumer — blocks here
            queue.task_done()

    await asyncio.gather(producer(), consumer())
```

### Detecting and Handling a Slow WebSocket Client

```python
import asyncio

async def stream_to_ws_safe(agent, prompt: str, websocket, timeout_seconds: float = 5.0):
    """
    Send tokens to a WebSocket client with a per-send timeout.
    Drops the connection cleanly if the client stops receiving.
    """
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                try:
                    await asyncio.wait_for(
                        websocket.send_json({"type": "token", "text": text}),
                        timeout=timeout_seconds
                    )
                except asyncio.TimeoutError:
                    await websocket.close(code=1001, reason="Consumer too slow")
                    return
```

### Agent Reuse Across Requests

Creating a `BedrockModel` and `Agent` on every request wastes boto3 session setup time. Create once at module level and reuse.

```python
from strands import Agent
from strands.models import BedrockModel

# Module-level singletons — created once at process startup
_model = BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                      region_name="us-east-1")
_agent = Agent(model=_model)

async def handle_request(user_id: str, prompt: str):
    # Clear conversation state between users (isolates history)
    _agent.conversation_manager.clear()
    async for event in _agent.stream_async(prompt):
        ...
```

### Limit Output Tokens for Low-Latency Responses

```python
from strands.models import BedrockModel

# Cap output for use cases where brevity is required
model = BedrockModel(
    model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
    region_name="us-east-1",
    max_tokens=256   # shorter responses arrive faster
)
```

---

## Testing Streaming Responses

### Unit Test: Collected Output

Mock `stream_async` to emit synthetic events and verify the consumer accumulates them correctly.

```python
import asyncio
import pytest
from unittest.mock import AsyncMock, patch

# Fake event sequence matching the real protocol
FAKE_EVENTS = [
    {"type": "message_start", "message": {"usage": {"input_tokens": 10, "output_tokens": 1}}},
    {"type": "content_block_start", "index": 0, "content_block": {"type": "text", "text": ""}},
    {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "Hello"}},
    {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": " world"}},
    {"type": "content_block_stop", "index": 0},
    {"type": "message_delta", "delta": {"stop_reason": "end_turn"}, "usage": {"output_tokens": 2}},
    {"type": "message_stop"},
]


async def fake_stream(*args, **kwargs):
    for event in FAKE_EVENTS:
        yield event


async def collect_text(agent, prompt: str) -> str:
    parts = []
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                parts.append(text)
    return "".join(parts)


@pytest.mark.asyncio
async def test_collect_text():
    from strands import Agent
    from strands.models import BedrockModel

    agent = Agent(model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                                     region_name="us-east-1"))

    with patch.object(agent, "stream_async", side_effect=fake_stream):
        result = await collect_text(agent, "Say hello")

    assert result == "Hello world"


@pytest.mark.asyncio
async def test_stream_emits_done_event():
    """Verify that the consumer receives a message_stop event."""
    from strands import Agent
    from strands.models import BedrockModel

    agent = Agent(model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                                     region_name="us-east-1"))

    events_seen = []
    with patch.object(agent, "stream_async", side_effect=fake_stream):
        async for event in agent.stream_async("test"):
            events_seen.append(event.get("type"))

    assert "message_stop" in events_seen
    assert events_seen.count("content_block_delta") == 2
```

### Integration Test: SSE Endpoint

```python
import pytest
from httpx import AsyncClient, ASGITransport
from unittest.mock import patch

# Assume `app` is the FastAPI application defined in your module
from myapp import app, agent, fake_stream


@pytest.mark.asyncio
async def test_sse_stream_endpoint():
    with patch.object(agent, "stream_async", side_effect=fake_stream):
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.get("/stream", params={"prompt": "hello"})

    assert response.status_code == 200
    assert "text/event-stream" in response.headers["content-type"]
    assert "Hello world" in response.text
    assert "event: done" in response.text
```

### Test: Sentence-Boundary Buffering

```python
@pytest.mark.asyncio
async def test_sentence_buffering():
    """Verify that sentences are flushed at boundaries, not per-token."""
    WORD_EVENTS = [
        {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "Hello world. "}},
        {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "How are "}},
        {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "you? "}},
        {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "Fine thanks"}},
        {"type": "message_stop"},
    ]

    async def fake(*a, **kw):
        for e in WORD_EVENTS:
            yield e

    flushed = []

    async def collect(s):
        if s is not None:
            flushed.append(s)

    from strands import Agent
    from strands.models import BedrockModel
    from unittest.mock import patch

    agent = Agent(model=BedrockModel(model_id="anthropic.claude-haiku-4-5-20251001-v1:0",
                                     region_name="us-east-1"))

    # Import the buffering function from your module
    from myapp import stream_by_sentence

    with patch.object(agent, "stream_async", side_effect=fake):
        await stream_by_sentence(agent, "prompt", collect)

    assert flushed[0] == "Hello world."
    assert flushed[1] == "How are you?"
    assert flushed[2] == "Fine thanks"
```

---

## Expanded Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `stream_async` called in sync context causes `RuntimeError: no running event loop` | `stream_async` is an async generator; must be used with `async for` inside an async function | Wrap in `asyncio.run(main())` or use `callback_handler` for sync code |
| All tokens arrive at once (no streaming effect) | Filtering on the wrong event type; or consuming the full generator before processing | Filter specifically for `event["type"] == "content_block_delta"` and process each event immediately |
| `KeyError: 'delta'` on some events | `delta` key is only present on `content_block_delta` events | Always use `event.get("delta", {})` — other event types do not carry this key |
| Stream ends with `stop_reason == "max_tokens"` instead of `"end_turn"` | Response was truncated by the model's token limit | Increase `max_tokens` in `BedrockModel(max_tokens=...)` |
| TTS audio lags significantly behind text | Buffering strategy creates chunks that are too large, causing long TTS synthesis time | Switch to sentence-boundary buffering; ensure TTS worker runs as a separate `asyncio.Task` |
| Memory grows unbounded during long streams | All tokens buffered before any processing | Process events immediately inside the `async for` loop; do not append all to a list |
| WebSocket client receives tokens out of order | Multiple concurrent `send_json` calls without serialization | Use a single writer task with a queue; never await `send_json` from more than one coroutine simultaneously |
| `tool_use` content block appears but tool input is empty | `input_json_delta` fragments not accumulated before `json.loads` | Build a `partial_json` buffer per block index; call `json.loads` only on `content_block_stop` |
| SSE stream buffered by nginx / proxy, tokens arrive in large batches | Proxy buffering enabled | Add `X-Accel-Buffering: no` response header; configure `proxy_buffering off` in nginx |
| `asyncio.CancelledError` during streaming | Client disconnected while stream is active | Wrap `async for` in `try/except asyncio.CancelledError` and clean up resources in `finally` |
| `ThrottlingException` mid-stream | Bedrock rate limit hit | Implement retry with exponential backoff around the full `stream_async` call; reduce concurrent streams |
| Callback handler fires on non-text events, corrupting output | Callback not checking event type | Always check `event.get("type") == "content_block_delta"` and `delta.get("type") == "text_delta"` before reading text |
| `stream_async` yields no events at all | Agent invoked with an empty prompt, or model returned an empty response | Verify prompt is non-empty; check `stop_reason` in `message_delta`; enable DEBUG logging with `logging.getLogger("strands").setLevel(logging.DEBUG)` |

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `stream_async` called in sync context | Wrap in `asyncio.run()` or use `callback_handler` |
| All tokens arrive at once | Filter for `content_block_delta` event type only |
| Final message not captured | Listen for `message_stop` for completion; `message_delta` for `stop_reason` |
| High memory usage | Process events immediately; do not buffer all tokens in a list |
| Tool input JSON is malformed | Accumulate `input_json_delta` fragments per block index; parse only on `content_block_stop` |
| WebSocket concurrent send corruption | Serialize all sends through a single writer task and queue |
| SSE tokens batched by proxy | Set `X-Accel-Buffering: no` header; configure `proxy_buffering off` in nginx |
| TTS audio blocked on LLM stream | Run TTS worker as a separate `asyncio.Task`; decouple with a bounded queue |
