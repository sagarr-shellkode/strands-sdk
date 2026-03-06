---
name: strands-sdk-streaming
description: Use when streaming agent responses token-by-token, handling StreamEvent callbacks, printing output as it arrives, integrating Strands streaming into a WebSocket or HTTP SSE endpoint, or using async generator response iteration. Triggers on: stream, stream_async, StreamEvent, callback_handler, yield tokens, real-time output, SSE.
---

# Streaming — Strands SDK

## Overview

Strands supports streaming responses so you can process tokens as they arrive instead of waiting for the full response. Use `stream_async` for async contexts or `callback_handler` for sync streaming.

## Quick Reference

| Method | Use case |
|---|---|
| `agent.stream_async(prompt)` | Async generator — yields events as they arrive |
| `callback_handler=fn` | Sync callback invoked per token/event |
| Event type `content_block_delta` | Contains the actual text tokens |
| Event type `message_stop` | Signals generation is complete |

## Async Streaming

```python
import asyncio
from strands import Agent
from strands.models import BedrockModel

agent = Agent(model=BedrockModel(...))

async def main():
    async for event in agent.stream_async("Explain neural networks."):
        if event.get("type") == "content_block_delta":
            delta = event.get("delta", {})
            if delta.get("type") == "text_delta":
                print(delta.get("text", ""), end="", flush=True)

asyncio.run(main())
```

## Sync Streaming via Callback

```python
from strands import Agent
from strands.models import BedrockModel

tokens = []

def on_token(event):
    if event.get("type") == "content_block_delta":
        text = event.get("delta", {}).get("text", "")
        tokens.append(text)
        print(text, end="", flush=True)

agent = Agent(model=BedrockModel(...), callback_handler=on_token)
agent("Write a poem about the ocean.")
```

## WebSocket / SSE Integration Pattern

```python
async def stream_to_websocket(websocket, prompt: str):
    agent = Agent(model=model)
    async for event in agent.stream_async(prompt):
        if event.get("type") == "content_block_delta":
            text = event.get("delta", {}).get("text", "")
            if text:
                await websocket.send_json({"type": "token", "text": text})
    await websocket.send_json({"type": "done"})
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| `stream_async` called in sync context | Wrap in `asyncio.run()` or use `callback_handler` |
| All tokens arrive at once | Filter for `content_block_delta` event type only |
| Final message not captured | Listen for `message_stop` event type for completion signal |
| High memory usage | Process events immediately, do not buffer all |
