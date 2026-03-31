# Chapter 4: Bootstrap And Startup

Bootstrap gets the harness from a cold process to a ready runtime. In Python, this is the difference between a script that technically runs and a harness that starts predictably.

## Why this system exists

Startup order decides what the harness trusts, what it knows, and what it exposes before the first user turn. Get it wrong and you create confusion before the loop even begins.

## Shared architecture

- load config and environment
- establish working directory and session state
- build the tool registry
- assemble instructions or system guidance
- initialize trust-sensitive integrations late
- prepare the model loop

## Python implementation

Use a bootstrap function that returns a single runtime object.

```python
from dataclasses import dataclass


@dataclass
class Runtime:
    client: object
    tool_registry: dict
    instructions: str
    session_store: object


def bootstrap() -> Runtime:
    ...
```

Keep instruction assembly out of ad hoc string-building in random files. It should be one startup responsibility with stable inputs.

## OpenAI Responses API mapping

Bootstrap is where you decide:

- which model to call through `responses.create`
- which tool definitions are exposed
- what `instructions` the runtime provides
- whether background work or streaming is available

Responses API does not remove bootstrap work. It simply gives the runtime a cleaner execution surface once startup is complete.

## Failure modes and tradeoffs

- eager initialization of untrusted plugins or MCP clients
- instruction assembly spread across the codebase
- startup side effects hidden in import time
- no single runtime object to pass into the query loop

## Build-it-yourself checklist

- centralize startup
- assemble one runtime object
- defer risky integrations
- generate instructions deterministically
- make tool exposure part of bootstrap, not prompt improvisation

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-4-bootstrap-startup)
- the original discovery work here focused on startup sequencing and prompt assembly
