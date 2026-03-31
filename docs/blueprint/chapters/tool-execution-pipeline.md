# Chapter 2 -- Tool Execution Pipeline

The tool system from Chapter 1 defines what the model can do. This chapter defines *how* those tools actually run -- the six-phase pipeline that turns a model-emitted tool call into a validated, permitted, scheduled execution with its result reinjected into the conversation.

## Why this system exists

The model asking for a tool is not the same thing as a tool being allowed to run. Without a structured pipeline, validation happens in random order, permission checks get skipped, parallel execution races against serial tools, and result formats drift between providers. One executor object owns all of this.

## The six-phase pipeline

Every tool call passes through six phases in strict order:

| Phase | Purpose | Fails to |
|-------|---------|----------|
| 1. **Validate** | Check inputs against `input_schema` | `ValidationError` |
| 2. **Semantic check** | Domain-specific input rules (path traversal, size limits) | `SemanticError` |
| 3. **Pre-hooks** | Transform or log before execution | Hook error |
| 4. **Permissions** | Authorization gate (`check_permissions`) | `PermissionError` |
| 5. **Execute** | Run the tool, serial or concurrent | `ExecutionError` |
| 6. **Post-hooks** | Auto-format output, record metrics | Hook error |

Phases 1--4 are fail-fast: any failure short-circuits the rest and returns an error result. Phase 5 is where concurrency decisions happen. Phase 6 runs even on failure (for logging).

## ToolResult -- frozen dataclass

Every tool execution returns the same shape. Frozen so nothing downstream can mutate a result after the fact.

```python
from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True)
class ToolResult:
    """Immutable result from any tool execution."""

    tool_name: str
    output: Any
    error: str | None = None
    duration_ms: float = 0.0
    was_concurrent: bool = False
```

## ToolExecutor -- the single execution path

```python
import asyncio
import time
import jsonschema
from dataclasses import dataclass
from typing import Any, Callable, Awaitable


@dataclass(frozen=True)
class HookContext:
    """Read-only snapshot passed to pre/post hooks."""

    tool_name: str
    payload: dict[str, Any]
    phase: str


# Type alias for hook callables
Hook = Callable[[HookContext], Awaitable[None]]


class ToolExecutor:
    """Six-phase pipeline that owns all tool execution."""

    def __init__(
        self,
        registry: "ToolRegistry",
        pre_hooks: list[Hook] | None = None,
        post_hooks: list[Hook] | None = None,
        on_progress: Callable[[str, float], None] | None = None,
    ) -> None:
        self._registry = registry
        self._pre_hooks = pre_hooks or []
        self._post_hooks = post_hooks or []
        self._on_progress = on_progress

    async def run(
        self,
        name: str,
        payload: dict[str, Any],
        context: "ToolContext",
    ) -> ToolResult:
        """Execute one tool call through all six phases."""
        start = time.monotonic()
        tool = self._registry.get(name)

        # Phase 1: Validate against JSON Schema
        try:
            jsonschema.validate(payload, tool.input_schema)
        except jsonschema.ValidationError as exc:
            return ToolResult(
                tool_name=name,
                output=None,
                error=f"Validation failed: {exc.message}",
            )

        # Phase 2: Semantic check (tool-specific rules)
        try:
            await tool.semantic_check(payload, context)
        except Exception as exc:
            return ToolResult(
                tool_name=name,
                output=None,
                error=f"Semantic check failed: {exc}",
            )

        # Phase 3: Pre-hooks
        hook_ctx = HookContext(
            tool_name=name, payload=payload, phase="pre"
        )
        for hook in self._pre_hooks:
            await hook(hook_ctx)

        # Phase 4: Permissions
        try:
            await tool.check_permissions(payload, context)
        except PermissionError as exc:
            return ToolResult(
                tool_name=name,
                output=None,
                error=f"Permission denied: {exc}",
            )

        # Phase 5: Execute
        try:
            self._report_progress(name, 0.0)
            result = await tool.execute(payload, context)
            self._report_progress(name, 1.0)
        except Exception as exc:
            result = ToolResult(
                tool_name=name,
                output=None,
                error=f"Execution failed: {exc}",
            )

        # Phase 6: Post-hooks (always run)
        post_ctx = HookContext(
            tool_name=name, payload=payload, phase="post"
        )
        for hook in self._post_hooks:
            await hook(post_ctx)

        elapsed = (time.monotonic() - start) * 1000
        return ToolResult(
            tool_name=name,
            output=result.output,
            error=result.error,
            duration_ms=elapsed,
        )

    def _report_progress(self, name: str, fraction: float) -> None:
        if self._on_progress is not None:
            self._on_progress(name, fraction)
```

## Concurrent vs serial execution

When the model emits multiple tool calls in one turn, the executor must decide: run them in parallel or serialize? The answer comes from `is_concurrency_safe()` on each tool.

```python
async def run_batch(
    executor: ToolExecutor,
    calls: list[tuple[str, dict[str, Any]]],
    context: "ToolContext",
    registry: "ToolRegistry",
) -> list[ToolResult]:
    """Run a batch of tool calls, parallel when safe."""
    parallel: list[tuple[str, dict[str, Any]]] = []
    serial: list[tuple[str, dict[str, Any]]] = []

    for name, payload in calls:
        tool = registry.get(name)
        if tool.is_concurrency_safe():
            parallel.append((name, payload))
        else:
            serial.append((name, payload))

    results: list[ToolResult] = []

    # Parallel-safe tools run under a TaskGroup
    if parallel:
        async with asyncio.TaskGroup() as tg:
            tasks = [
                tg.create_task(executor.run(name, payload, context))
                for name, payload in parallel
            ]
        results.extend(task.result() for task in tasks)

    # Serial tools run one at a time, in order
    for name, payload in serial:
        result = await executor.run(name, payload, context)
        results.append(result)

    return results
```

`asyncio.TaskGroup` (Python 3.11+) is the right primitive here. If any parallel tool raises an unhandled exception, the group cancels siblings and propagates -- no orphaned coroutines.

## Progress callbacks for long-running tools

Tools like `bash` or `search` can take seconds. The `on_progress` callback lets the UI layer show incremental feedback without coupling tool logic to display code.

```python
def terminal_progress(tool_name: str, fraction: float) -> None:
    """Example progress callback for terminal UI."""
    bar_width = 30
    filled = int(bar_width * fraction)
    bar = "#" * filled + "-" * (bar_width - filled)
    print(f"\r  {tool_name} [{bar}] {fraction:.0%}", end="", flush=True)
    if fraction >= 1.0:
        print()


executor = ToolExecutor(
    registry=registry,
    on_progress=terminal_progress,
)
```

For finer-grained progress inside a tool, the tool itself can accept the callback through the context and call it at intermediate steps.

## Reinjection -- feeding results back to the model

After execution, every result must be sent back to the model in its provider-specific format. The two SDKs differ in shape but not in concept: the model needs the tool call ID paired with the output content.

=== "Anthropic"

    Tool results go back as `tool_result` content blocks inside a `user` message. Each block references the `tool_use_id` from the model's response.

    ```python
    import anthropic

    client = anthropic.Anthropic()

    def build_tool_result_message(
        tool_use_id: str,
        result: ToolResult,
    ) -> dict:
        """Build an Anthropic tool_result content block."""
        if result.error:
            return {
                "type": "tool_result",
                "tool_use_id": tool_use_id,
                "is_error": True,
                "content": result.error,
            }
        return {
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": str(result.output),
        }

    # Full tool loop
    async def anthropic_tool_loop(
        messages: list[dict],
        registry: "ToolRegistry",
        executor: ToolExecutor,
        context: "ToolContext",
    ) -> str:
        while True:
            response = client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                messages=messages,
                tools=registry.to_anthropic_tools(),
            )

            if response.stop_reason != "tool_use":
                # Model is done calling tools
                return response.content[0].text

            # Collect tool_use blocks and execute them
            tool_uses = [
                b for b in response.content if b.type == "tool_use"
            ]
            calls = [(b.name, b.input) for b in tool_uses]
            results = await run_batch(
                executor, calls, context, registry
            )

            # Build result blocks and continue
            result_blocks = [
                build_tool_result_message(tu.id, r)
                for tu, r in zip(tool_uses, results)
            ]
            messages.append(
                {"role": "assistant", "content": response.content}
            )
            messages.append(
                {"role": "user", "content": result_blocks}
            )
    ```

=== "OpenAI"

    Tool results go back as `function_call_output` items in the next `input` list. Each item references the `call_id` from the model's `function_call` output.

    ```python
    import json
    from openai import OpenAI

    client = OpenAI()

    def build_function_output(
        call_id: str,
        result: ToolResult,
    ) -> dict:
        """Build an OpenAI function_call_output item."""
        output = (
            result.error
            if result.error
            else json.dumps(result.output)
        )
        return {
            "type": "function_call_output",
            "call_id": call_id,
            "output": output,
        }

    # Full tool loop
    async def openai_tool_loop(
        input_items: list[dict],
        registry: "ToolRegistry",
        executor: ToolExecutor,
        context: "ToolContext",
    ) -> str:
        while True:
            response = client.responses.create(
                model="gpt-4.1",
                input=input_items,
                tools=registry.to_openai_tools(),
            )

            fn_calls = [
                item for item in response.output
                if item.type == "function_call"
            ]
            if not fn_calls:
                # Model is done calling tools
                text_items = [
                    item for item in response.output
                    if item.type == "message"
                ]
                return text_items[0].content[0].text

            # Execute all function calls
            calls = [
                (fc.name, json.loads(fc.arguments))
                for fc in fn_calls
            ]
            results = await run_batch(
                executor, calls, context, registry
            )

            # Reinject results and continue
            input_items = [
                *input_items,
                *[item.model_dump() for item in response.output],
                *[
                    build_function_output(fc.call_id, r)
                    for fc, r in zip(fn_calls, results)
                ],
            ]
    ```

## Tool result format comparison

| Aspect | Anthropic | OpenAI |
|--------|-----------|--------|
| Container | `tool_result` content block in `user` message | `function_call_output` item in `input` list |
| ID field | `tool_use_id` (matches `tool_use` block) | `call_id` (matches `function_call` item) |
| Error flag | `"is_error": True` with error in `content` | Error string in `output` (no dedicated flag) |
| Content type | String or list of content blocks | JSON string |
| Batching | Multiple `tool_result` blocks in one `user` message | Multiple `function_call_output` items in `input` |

## Build it yourself

1. Define a frozen `ToolResult` dataclass -- one shape for every tool, every provider.
2. Build a `ToolExecutor` with the six phases in strict order: validate, semantic check, pre-hooks, permissions, execute, post-hooks.
3. Split incoming tool call batches by `is_concurrency_safe()` -- parallel-safe tools run under `asyncio.TaskGroup`, serial tools run sequentially.
4. Wire a progress callback into the executor so the UI can show feedback without coupling to tool internals.
5. Write one `build_tool_result_message` (Anthropic) and one `build_function_output` (OpenAI) function that maps `ToolResult` to the provider format.
6. Implement the tool loop: send messages, check for tool calls in the response, execute through the pipeline, reinject results, repeat until the model stops requesting tools.
