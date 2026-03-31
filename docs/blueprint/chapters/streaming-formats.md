# Streaming Formats -- Anthropic vs OpenAI

Both providers use Server-Sent Events (SSE) but with different event types, delta structures, and tool-call handling. Here is how to handle both.

## Why streaming matters

- Users see output as it generates (lower perceived latency)
- Tool calls can start executing before the full response arrives
- Budget and abort checks can happen mid-stream

## SSE event lifecycle

=== "Anthropic"

    ```
    event: message_start       -> message metadata, usage
    event: content_block_start -> new text/tool_use block beginning
    event: content_block_delta -> incremental text or tool input JSON
    event: content_block_stop  -> block complete
    event: message_delta       -> stop_reason, final usage
    event: message_stop        -> stream done
    ```

=== "OpenAI"

    ```
    event: response.created            -> response metadata
    event: response.output_item.added  -> new output item starting
    event: response.content_part.added -> content part starting
    event: response.output_text.delta  -> incremental text
    event: response.function_call_arguments.delta -> tool args streaming
    event: response.output_item.done   -> item complete
    event: response.completed          -> stream done
    ```

## Handling streams in Python

=== "Anthropic"

    ```python
    import anthropic

    client = anthropic.Anthropic()

    with client.messages.stream(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[{"role": "user", "content": "Hello"}],
        tools=tools,
    ) as stream:
        for event in stream:
            match event.type:
                case "message_start":
                    print(f"[Model: {event.message.model}]")
                case "content_block_start":
                    if event.content_block.type == "tool_use":
                        print(f"\n[Tool call: {event.content_block.name}]")
                case "content_block_delta":
                    if event.delta.type == "text_delta":
                        print(event.delta.text, end="", flush=True)
                    elif event.delta.type == "input_json_delta":
                        pass  # accumulate tool input JSON
                case "message_delta":
                    print(f"\n[Stop: {event.delta.stop_reason}]")
                    print(f"[Usage: {event.usage}]")
                case "message_stop":
                    print("\n[Stream complete]")
    ```

=== "OpenAI"

    ```python
    from openai import OpenAI

    client = OpenAI()

    stream = client.responses.create(
        model="gpt-4.1",
        input="Hello",
        tools=tools,
        stream=True,
    )

    for event in stream:
        match event.type:
            case "response.created":
                print(f"[Response: {event.response.id}]")
            case "response.output_text.delta":
                print(event.delta, end="", flush=True)
            case "response.function_call_arguments.delta":
                pass  # accumulate tool arguments
            case "response.output_item.done":
                if event.item.type == "function_call":
                    print(f"\n[Tool: {event.item.name}({event.item.arguments})]")
            case "response.completed":
                print(f"\n[Usage: {event.response.usage}]")
    ```

## Side-by-side comparison

| Concept | Anthropic Event | OpenAI Event |
|---------|----------------|--------------|
| Stream start | `message_start` | `response.created` |
| Text chunk | `content_block_delta` (text_delta) | `response.output_text.delta` |
| Tool call start | `content_block_start` (tool_use) | `response.output_item.added` (function_call) |
| Tool args chunk | `content_block_delta` (input_json_delta) | `response.function_call_arguments.delta` |
| Tool call complete | `content_block_stop` | `response.output_item.done` |
| Stop reason | `message_delta` (stop_reason) | `response.completed` |
| Usage | `message_delta` (usage) | `response.completed` (usage) |
| Stream end | `message_stop` | `response.completed` |

## Unified stream handler

When supporting both providers, normalize their events into a common type
to keep downstream logic provider-agnostic.

```python
from dataclasses import dataclass, field
from typing import AsyncIterator, Literal


@dataclass(frozen=True)
class StreamEvent:
    """Provider-agnostic streaming event."""

    type: Literal["text", "tool_start", "tool_delta", "tool_done", "usage", "done"]
    text: str = ""
    tool_name: str = ""
    tool_id: str = ""
    tool_args: str = ""
    usage: dict = field(default_factory=dict)


async def normalize_anthropic_stream(stream) -> AsyncIterator[StreamEvent]:
    """Convert Anthropic SSE events to unified StreamEvents."""
    for event in stream:
        match event.type:
            case "content_block_delta":
                if event.delta.type == "text_delta":
                    yield StreamEvent(type="text", text=event.delta.text)
                elif event.delta.type == "input_json_delta":
                    yield StreamEvent(
                        type="tool_delta", tool_args=event.delta.partial_json
                    )
            case "content_block_start":
                if event.content_block.type == "tool_use":
                    yield StreamEvent(
                        type="tool_start",
                        tool_name=event.content_block.name,
                        tool_id=event.content_block.id,
                    )
            case "content_block_stop":
                yield StreamEvent(type="tool_done")
            case "message_delta":
                yield StreamEvent(
                    type="usage",
                    usage={"stop_reason": event.delta.stop_reason},
                )
            case "message_stop":
                yield StreamEvent(type="done")


async def normalize_openai_stream(stream) -> AsyncIterator[StreamEvent]:
    """Convert OpenAI SSE events to unified StreamEvents."""
    for event in stream:
        match event.type:
            case "response.output_text.delta":
                yield StreamEvent(type="text", text=event.delta)
            case "response.output_item.added":
                if event.item.type == "function_call":
                    yield StreamEvent(
                        type="tool_start",
                        tool_name=event.item.name,
                        tool_id=event.item.call_id,
                    )
            case "response.function_call_arguments.delta":
                yield StreamEvent(type="tool_delta", tool_args=event.delta)
            case "response.output_item.done":
                if event.item.type == "function_call":
                    yield StreamEvent(
                        type="tool_done", tool_args=event.item.arguments
                    )
            case "response.completed":
                yield StreamEvent(type="done", usage=vars(event.response.usage))
```

## Tool calls during streaming

Tool-use blocks can start streaming while text is still being generated.
The pattern below starts tool execution as each block completes, without
waiting for the full response to finish.

```python
import json

pending_tools: dict[str, dict] = {}
tool_results: list = []

async for event in normalize_stream(raw_stream):
    match event.type:
        case "tool_start":
            pending_tools[event.tool_id] = {
                "name": event.tool_name,
                "args": "",
            }
        case "tool_delta":
            for tool_id in pending_tools:
                pending_tools[tool_id]["args"] += event.tool_args
        case "tool_done":
            # Tool is complete -- execute immediately, do not wait for stream end
            tool_id = next(iter(pending_tools), None)
            if tool_id:
                tool_info = pending_tools.pop(tool_id)
                result = await executor.run(
                    tool_info["name"], json.loads(tool_info["args"])
                )
                tool_results.append(result)
        case "text":
            print(event.text, end="", flush=True)
```

## Build-it-yourself checklist

1. Pick your SDK(s) and handle their SSE event types
2. Normalize to a common `StreamEvent` type if supporting both providers
3. Execute tools as they complete during streaming (do not wait for stream end)
