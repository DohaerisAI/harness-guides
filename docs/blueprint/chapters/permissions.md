# Chapter 5: Permission System

Permissions are what keep a harness from becoming a vague yes-machine. In Python, this chapter is about creating one inspectable policy layer instead of hiding safety decisions inside each tool.

## Why this system exists

You cannot reason about a harness if "can this run?" depends on whichever function happens to be holding the payload.

## Shared architecture

- permission context combines runtime rules
- validation answers "is this payload well formed?"
- permissions answer "is this allowed now?"
- shell or filesystem actions need stricter policy than read-only inspection
- the user experience should reflect allow, deny, and ask clearly

## Python implementation

Model permission state explicitly.

```python
from dataclasses import dataclass, field


@dataclass
class PermissionContext:
    allow_paths: list[str] = field(default_factory=list)
    deny_paths: list[str] = field(default_factory=list)
    shell_mode: str = "ask"
```

Then write shared helpers like:

- `check_read_permission(path, context)`
- `check_write_permission(path, context)`
- `classify_shell_command(command, context)`

Keep those helpers outside the tool bodies.

## OpenAI Responses API mapping

Responses API may let the model request a tool, but it does not decide whether your runtime should execute it. The harness still owns the permission decision before any local tool or external action runs.

Use the model for reasoning. Keep authorization local.

## Failure modes and tradeoffs

- tools bypass central permission helpers
- validation errors are treated like permission denials
- shell safety is left to prompt instructions alone
- allow/deny state is not persisted or inspectable

## Build-it-yourself checklist

- define one permission context
- separate malformed input from disallowed behavior
- add shared read/write/shell helpers
- make permission outcomes visible to the loop
- never let the model be the final authority on local execution

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-5-permission-system)
- the source discovery here came from the permission decision chain, shell-security notes, and the minimal Python model
