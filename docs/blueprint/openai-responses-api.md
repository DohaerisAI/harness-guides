# OpenAI Responses API

Harness Guides uses OpenAI's Responses API as the default OpenAI reference surface.

Official docs:

- [Responses API overview](https://developers.openai.com/api/reference/responses/overview)
- [Responses guide](https://platform.openai.com/docs/guides/responses)
- [Tools guide](https://platform.openai.com/docs/guides/tools)

## Why this is the default here

The Responses API is the right fit for harness documentation because it brings model output, tool use, streaming, long-running work, and conversation continuity into one surface. That maps much more cleanly to agent harness design than teaching everything through legacy chat-completions patterns.

## Concepts used across this site

- `responses.create` for the main loop entrypoint
- `input` for user and system-side content
- `instructions` for high-level runtime guidance
- `previous_response_id` for response continuity
- `tools` for built-in tools, remote MCP, and custom function calls
- `tool_choice` when the harness wants tighter control
- `parallel_tool_calls` when the runtime allows safe concurrency
- `stream` for incremental model output and tool events
- `background` for long-running work that should outlive one synchronous wait
- `include` when the runtime needs richer returned data

## Python example

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-5",
    instructions="You are a coding harness. Use tools when needed.",
    input="Summarize the repository layout and call tools if necessary.",
    tools=[
        {
            "type": "function",
            "name": "read_file",
            "description": "Read a file from disk",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string"},
                },
                "required": ["path"],
            },
        }
    ],
    parallel_tool_calls=True,
)
```

## How to use this page

- Read it before the query loop and tool chapters.
- Use it as the vocabulary guide for every OpenAI-specific example.
- Keep architecture decisions provider-portable, even when the examples use Responses API.
