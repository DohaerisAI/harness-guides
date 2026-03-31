# Chapter 1 -- Tool System

The tool system defines what the model can do -- every capability is a self-contained module with a schema, permissions, and execution logic.

## Why this system exists

Tools are the hands of the agent. Without them it can only produce text. If the tool contract is vague, everything above it gets worse: permissions become ad hoc, execution order becomes unclear, and model behavior drifts because the runtime has no stable capability boundary.

## Shared architecture

Every tool implements a common interface:

| Member | Purpose |
|--------|---------|
| `name` | Unique identifier |
| `description` | What it does (for the model) |
| `input_schema` | JSON Schema for parameters |
| `execute(payload, context)` | The actual work |
| `check_permissions(payload, context)` | Authorization gate |
| `is_read_only()` | Safe for parallel execution? |
| `is_concurrency_safe()` | Can run alongside other tools? |

## Python implementation

### Core types

```python
from pydantic import BaseModel
from typing import Any, Protocol


class ToolContext(BaseModel):
    """Immutable execution context passed to every tool call."""

    working_directory: str
    session_id: str
    model_config = {"frozen": True}


class ToolResult(BaseModel):
    """Immutable result returned by every tool execution."""

    output: Any
    error: str | None = None
    duration_ms: float = 0
    model_config = {"frozen": True}


class Tool(Protocol):
    """Protocol that every tool must satisfy."""

    name: str
    description: str
    input_schema: dict[str, Any]

    async def execute(
        self, payload: dict[str, Any], context: ToolContext
    ) -> ToolResult: ...

    async def check_permissions(
        self, payload: dict[str, Any], context: ToolContext
    ) -> None: ...

    def is_read_only(self) -> bool: ...

    def is_concurrency_safe(self) -> bool: ...
```

### ToolRegistry with both SDK formats

The registry is the single source of truth. Tool definitions for any provider are generated from it -- never hand-assembled.

```python
class ToolRegistry:
    def __init__(self) -> None:
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> "ToolRegistry":
        """Register a tool. Returns a new registry (immutable pattern)."""
        new_reg = ToolRegistry()
        new_reg._tools = {**self._tools, tool.name: tool}
        return new_reg

    def get(self, name: str) -> Tool:
        if name not in self._tools:
            raise KeyError(f"Unknown tool: {name}")
        return self._tools[name]

    def to_anthropic_tools(self) -> list[dict]:
        """Emit tool definitions for the Anthropic Messages API."""
        return [
            {
                "name": tool.name,
                "description": tool.description,
                "input_schema": tool.input_schema,
            }
            for tool in self._tools.values()
        ]

    def to_openai_tools(self) -> list[dict]:
        """Emit tool definitions for the OpenAI Responses API."""
        return [
            {
                "type": "function",
                "name": tool.name,
                "description": tool.description,
                "parameters": tool.input_schema,
            }
            for tool in self._tools.values()
        ]
```

### Reference implementation: ReadFileTool

```python
import aiofiles
import os
import time


class ReadFileTool:
    name = "read_file"
    description = "Read the contents of a file from disk."
    input_schema = {
        "type": "object",
        "properties": {
            "file_path": {
                "type": "string",
                "description": "Absolute path to the file to read.",
            },
            "offset": {
                "type": "integer",
                "description": "Line number to start reading from (1-based).",
            },
            "limit": {
                "type": "integer",
                "description": "Maximum number of lines to read.",
            },
        },
        "required": ["file_path"],
    }

    async def execute(
        self, payload: dict, context: ToolContext
    ) -> ToolResult:
        start = time.monotonic()
        path = payload["file_path"]
        offset = payload.get("offset", 1)
        limit = payload.get("limit", 2000)

        async with aiofiles.open(path) as f:
            lines = await f.readlines()

        selected = lines[offset - 1 : offset - 1 + limit]
        numbered = [
            f"{i + offset}\t{line}" for i, line in enumerate(selected)
        ]

        return ToolResult(
            output="".join(numbered),
            duration_ms=(time.monotonic() - start) * 1000,
        )

    async def check_permissions(
        self, payload: dict, context: ToolContext
    ) -> None:
        path = payload["file_path"]
        if not os.path.isabs(path):
            raise PermissionError("File path must be absolute")
        if not path.startswith(context.working_directory):
            raise PermissionError(
                f"Path outside working directory: {path}"
            )

    def is_read_only(self) -> bool:
        return True

    def is_concurrency_safe(self) -> bool:
        return True
```

## Using tools with both SDKs

=== "Anthropic"

    ```python
    import anthropic
    import asyncio

    client = anthropic.Anthropic()

    registry = ToolRegistry()
    registry = registry.register(ReadFileTool())

    context = ToolContext(
        working_directory="/tmp",
        session_id="session-001",
    )

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[
            {"role": "user", "content": "Read the file at /tmp/test.py"}
        ],
        tools=registry.to_anthropic_tools(),
    )

    # Handle tool_use blocks in response.content
    for block in response.content:
        if block.type == "tool_use":
            tool = registry.get(block.name)
            await tool.check_permissions(block.input, context)
            result = await tool.execute(block.input, context)
            # Send tool_result back to continue the conversation
    ```

=== "OpenAI"

    ```python
    import json
    from openai import OpenAI

    client = OpenAI()

    registry = ToolRegistry()
    registry = registry.register(ReadFileTool())

    context = ToolContext(
        working_directory="/tmp",
        session_id="session-001",
    )

    response = client.responses.create(
        model="gpt-4.1",
        input="Read the file at /tmp/test.py",
        tools=registry.to_openai_tools(),
    )

    # Handle function_call items in response.output
    for item in response.output:
        if item.type == "function_call":
            tool = registry.get(item.name)
            payload = json.loads(item.arguments)
            await tool.check_permissions(payload, context)
            result = await tool.execute(payload, context)
            # Send function_call_output back to continue
    ```

## `build_tool()` factory pattern

A factory that applies fail-closed defaults so that forgetting a flag never opens a hole.

```python
from typing import Any, Callable, Awaitable


TOOL_DEFAULTS: dict[str, Any] = {
    "is_read_only": staticmethod(lambda: False),
    "is_concurrency_safe": staticmethod(lambda: False),
    "check_permissions": staticmethod(
        lambda payload, context: None
    ),
}


def build_tool(definition: dict[str, Any]) -> Tool:
    """Create a tool from a dict, applying fail-closed defaults."""
    merged = {**TOOL_DEFAULTS, **definition}
    return type("DynamicTool", (), merged)()
```

Usage:

```python
search_tool = build_tool({
    "name": "search_code",
    "description": "Search for a pattern across files.",
    "input_schema": {
        "type": "object",
        "properties": {
            "pattern": {"type": "string"},
            "glob": {"type": "string"},
        },
        "required": ["pattern"],
    },
    "execute": my_search_impl,
    "is_read_only": lambda: True,
    "is_concurrency_safe": lambda: True,
})
```

## Tool pool assembly

```python
def assemble_tool_pool(
    built_in: list[Tool],
    mcp_tools: list[Tool],
    deny_rules: set[str],
) -> list[Tool]:
    """Merge built-in + MCP tools, filter by deny rules,
    sort for prompt-cache stability."""
    allowed_builtin = [t for t in built_in if t.name not in deny_rules]
    allowed_mcp = [t for t in mcp_tools if t.name not in deny_rules]

    # Sort each partition for prompt-cache stability
    allowed_builtin.sort(key=lambda t: t.name)
    allowed_mcp.sort(key=lambda t: t.name)

    # Built-ins first (cache prefix), then MCP
    combined = allowed_builtin + allowed_mcp

    # Deduplicate by name (built-in wins)
    seen: set[str] = set()
    result: list[Tool] = []
    for tool in combined:
        if tool.name not in seen:
            seen.add(tool.name)
            result.append(tool)
    return result
```

## Key differences: Anthropic vs OpenAI tool format

| Field | Anthropic | OpenAI |
|-------|-----------|--------|
| Wrapper | `{"name", "description", "input_schema"}` | `{"type": "function", "name", "description", "parameters"}` |
| Schema key | `input_schema` | `parameters` |
| Type field | Not needed | `"type": "function"` required |
| Tool result | `tool_result` content block | `function_call_output` in input |

The `ToolRegistry.to_anthropic_tools()` and `to_openai_tools()` methods handle this translation so application code never deals with format differences directly.

## Build it yourself

1. Define one shared `Tool` protocol with `name`, `description`, `input_schema`, `execute`, `check_permissions`, and behavior flags.
2. Build a `ToolRegistry` that emits provider-specific tool arrays from a single source of truth.
3. Use `build_tool()` with fail-closed defaults so omitting a flag never opens a security hole.
4. Assemble the final tool pool with deterministic ordering for prompt-cache stability.
5. Wire the tool loop: parse `tool_use` / `function_call` from the response, dispatch through the registry, and return results.
