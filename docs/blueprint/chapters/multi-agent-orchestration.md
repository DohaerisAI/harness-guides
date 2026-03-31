# Chapter 7: Multi-Agent Orchestration

Sub-agents are where harnesses stop being single-loop tools and become orchestration systems. In Python, this means explicit delegation boundaries, not just spawning more loops because it feels powerful.

## Why this system exists

Delegation only helps when workers have bounded tasks and the parent loop retains control. Otherwise multi-agent systems become expensive confusion amplifiers.

## Shared architecture

- parent loop owns user intent
- child workers get bounded tasks
- shared context must be intentionally filtered
- results must come back in a shape the parent can synthesize
- permission and cost policy still belong to the parent runtime

## Python implementation

Represent a delegated job explicitly.

```python
from dataclasses import dataclass


@dataclass
class WorkerTask:
    objective: str
    allowed_tools: list[str]
    return_format: str
```

Pass only what the worker needs. Do not dump the entire parent runtime state into every child by default.

## OpenAI Responses API mapping

Responses API can power both the parent and delegated worker calls, but the orchestration rules stay local:

- one worker request per bounded task
- one return shape per worker type
- parent decides synthesis and follow-up

If you use background work, treat it as controlled long-running delegation, not as a replacement for orchestration design.

## Failure modes and tradeoffs

- spawning workers without bounded ownership
- duplicating work across parent and child
- letting child prompts drift too far from the parent task
- losing cost visibility across delegated calls

## Build-it-yourself checklist

- define worker task objects
- keep the parent as source of truth
- constrain worker tool access
- standardize worker result shapes
- add explicit cost and permission accounting for sub-agents

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-7-multi-agent-orchestration)
- the source discovery here came from sub-agent spawning, coordinator mode, and cache-sharing rules
