# Chapter 1: Tool System

Tools are the harness surface that turns a model into an actor. In the Python-first version of this guide, the tool system exists to make capabilities explicit, typed, permissionable, and easy to schedule from the main loop.

## Why this system exists

If the tool contract is vague, everything above it gets worse. Permissions become ad hoc, execution order becomes unclear, and model behavior drifts because the runtime has no stable capability boundary.

## Shared architecture

- a tool has identity, input shape, output shape, and behavior flags
- validation is separate from permission checks
- tool registration is deterministic
- concurrency safety is explicit instead of assumed
- prompt and tool stability matters because it affects model behavior and caching

## Python implementation

Use a small Python protocol or abstract base class for tools, plus typed payload models.

```python
from dataclasses import dataclass
from typing import Any, Protocol


@dataclass
class ToolContext:
    working_directory: str
    session_id: str


class Tool(Protocol):
    name: str

    async def validate_input(self, payload: dict[str, Any]) -> None: ...
    async def check_permissions(self, payload: dict[str, Any], context: ToolContext) -> None: ...
    async def call(self, payload: dict[str, Any], context: ToolContext) -> dict[str, Any]: ...

    def is_read_only(self) -> bool: ...
    def is_concurrency_safe(self) -> bool: ...
```

Practical defaults:

- unknown tools should default to non-concurrency-safe
- mutability should be explicit
- new tools should not bypass shared permission helpers
- registration order should be deterministic before passing tools to the model layer

## OpenAI Responses API mapping

In Responses API terms, the tool system becomes the source of truth for the `tools` array you pass to `responses.create`. Each tool definition should be generated from one internal registry, not hand-assembled in multiple places.

```python
response = client.responses.create(
    model="gpt-5",
    input="Find the largest Python modules in this repo.",
    tools=tool_registry.to_openai_tools(),
    parallel_tool_calls=True,
)
```

## Failure modes and tradeoffs

- tool schemas drift from implementation
- permission logic gets reimplemented inside tools
- unsafe tools are accidentally marked parallel
- tool ordering becomes unstable between requests

## Build-it-yourself checklist

- define one shared tool contract
- separate validation from permission policy
- centralize tool registration
- emit OpenAI tool definitions from the same source of truth
- fail closed on concurrency

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-1-tool-system)
- the original discovery work here came from the TypeScript tool interface, factory defaults, and registration pipeline
