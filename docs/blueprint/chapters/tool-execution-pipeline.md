# Chapter 2: Tool Execution Pipeline

This chapter explains the path from a model-emitted tool call to actual execution. In Python, this usually means an explicit `asyncio` pipeline with one place that performs validation, permission checks, execution scheduling, and result reinjection.

## Why this system exists

The model asking for a tool is not the same thing as a tool being allowed to run. A good execution pipeline turns that request into a stable sequence of checks and events.

## Shared architecture

- parse requested tool call
- validate input
- apply permission policy
- schedule based on concurrency safety
- stream progress if needed
- inject tool results back into the loop

## Python implementation

Use one executor object that owns concurrency rules instead of letting each tool improvise execution behavior.

```python
class ToolExecutor:
    def __init__(self, registry):
        self.registry = registry

    async def run_tool_call(self, name: str, payload: dict, context):
        tool = self.registry[name]
        await tool.validate_input(payload)
        await tool.check_permissions(payload, context)
        return await tool.call(payload, context)
```

Add a scheduler layer on top when some tools are safe in parallel and others must run exclusively.

## OpenAI Responses API mapping

Responses API gives you tool requests through the unified response surface. The harness should translate those into executor calls, then send the tool outputs back into the next `responses.create` call or continued response flow.

Focus areas:

- `parallel_tool_calls=True` only when your executor can enforce safe concurrency
- `stream=True` when you want incremental model/tool progress
- `background=True` when the work should continue outside one foreground wait

## Failure modes and tradeoffs

- parallel execution without explicit safety metadata
- validation and permission checks happening in different orders across tools
- long-running tools without progress or background handling
- tool results reinjected in inconsistent shapes

## Build-it-yourself checklist

- build one executor layer
- standardize the execution order
- separate concurrency policy from tool logic
- define one result shape for reinjection
- use Responses API parallel/background features only when the runtime can honor them

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-2-tool-execution-pipeline)
- the original discovery material here centered on the six-phase execution pipeline and streaming executor model
