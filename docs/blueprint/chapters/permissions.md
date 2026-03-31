# Chapter 5 — Permission System

Permissions exist because an AI harness that can read files, write code, and execute
shell commands is dangerous without a single, inspectable gate between "the model wants
to do X" and "the runtime actually does X." Centralising that gate in one policy layer
— rather than scattering safety checks across individual tools — makes the system
auditable, testable, and easy to override per-project.

---

## The Allow / Deny / Ask Enum

Every permission decision resolves to exactly one of three outcomes:

| Verdict | Meaning |
|---------|---------|
| **Allow** | Execute without prompting the user. |
| **Deny** | Refuse and return an error to the model. |
| **Ask** | Pause execution and present the action to the user for approval. |

---

## Rule Sources (Priority Order)

Rules are evaluated top-down; the first match wins:

1. **Policy** — hard-coded safety rails (e.g. never `rm -rf /`)
2. **Project** — `.claude/settings.json` in the repo root
3. **User** — `~/.claude/settings.json`
4. **CLI flags** — `--allow-edit`, `--dangerously-skip-permissions`
5. **Session** — runtime grants the user gave during this session

---

## Path-Based Glob Matching

File operations are checked against glob patterns attached to each rule.
Patterns follow `fnmatch` semantics with `**` for recursive descent:

| Pattern | Matches |
|---------|---------|
| `src/**/*.py` | Any Python file under `src/` |
| `*.env` | Dotenv files in the working directory |
| `/etc/**` | Anything under `/etc` (absolute) |

A rule carries a list of globs and a verdict. The first rule whose glob matches
the requested path determines the outcome.

---

## Shell Command Classification

Shell commands are classified by a dedicated subsystem that parses the command
into an AST (via tree-sitter, never regex) and flags risk signals. A brief
overview of the categories it detects:

- **QuoteContext** — commands hidden inside quotes or heredocs
- **CompoundStructure** — pipes, subshells, command substitution
- **DangerousPatterns** — `rm -rf`, `chmod 777`, `curl | sh`, etc.

The classifier outputs a risk score that feeds into the permission decision chain.
For the full deep-dive, see [Chapter 12 — YOLO Classifier](yolo-classifier.md).

---

## Permission Decision Chain

```
Tool call requested
        |
  ┌─────▼──────┐
  │ PreToolUse  │  Hook can Allow / Deny / Skip
  │   Hooks     │
  └─────┬──────┘
        │ (no hook verdict)
  ┌─────▼──────┐
  │  Rule scan  │  Policy → Project → User → CLI → Session
  │  (globs +   │  First matching rule wins
  │   commands) │
  └─────┬──────┘
        │ (no rule matched)
  ┌─────▼──────┐
  │ Classifier  │  Side-query to fast model (Haiku)
  │ (YOLO mode) │  Returns Allow or Deny + reason
  └─────┬──────┘
        │ (classifier skipped or Ask)
  ┌─────▼──────┐
  │  User       │  Terminal prompt: allow / deny / always-allow
  │  Dialog     │
  └─────┴──────┘
```

---

## Python Implementation

```python
from __future__ import annotations

import fnmatch
from dataclasses import dataclass, field
from enum import Enum, auto
from typing import Sequence


class Verdict(Enum):
    ALLOW = auto()
    DENY = auto()
    ASK = auto()


@dataclass(frozen=True)
class PermissionRule:
    """A single rule: a set of glob patterns mapped to a verdict."""
    globs: tuple[str, ...]
    verdict: Verdict
    reason: str = ""


@dataclass(frozen=True)
class PermissionPolicy:
    """Immutable, ordered rule set evaluated top-down."""
    rules: tuple[PermissionRule, ...] = ()

    # -- construction helpers (return new instances) -----------------------

    def with_rule(self, rule: PermissionRule) -> PermissionPolicy:
        return PermissionPolicy(rules=(*self.rules, rule))

    def with_rules(self, rules: Sequence[PermissionRule]) -> PermissionPolicy:
        return PermissionPolicy(rules=(*self.rules, *rules))

    # -- evaluation --------------------------------------------------------

    def check_path(self, path: str) -> Verdict:
        """Return the verdict for *path*, or ASK if no rule matches."""
        for rule in self.rules:
            if any(fnmatch.fnmatch(path, g) for g in rule.globs):
                return rule.verdict
        return Verdict.ASK

    def check_command(self, command: str, risk_score: float) -> Verdict:
        """Classify a shell command using the attached risk score.

        Commands above the danger threshold are denied; low-risk
        commands are allowed; everything else goes to ASK.
        """
        if risk_score >= 0.8:
            return Verdict.DENY
        if risk_score <= 0.2:
            return Verdict.ALLOW
        return Verdict.ASK


@dataclass(frozen=True)
class PermissionDecision:
    """The final, auditable result of a permission check."""
    verdict: Verdict
    source: str          # e.g. "policy", "project", "user", "session"
    matched_rule: PermissionRule | None = None
    reason: str = ""


def evaluate(
    layers: Sequence[tuple[str, PermissionPolicy]],
    path: str,
) -> PermissionDecision:
    """Walk rule sources in priority order; first match wins."""
    for source_name, policy in layers:
        verdict = policy.check_path(path)
        if verdict is not Verdict.ASK:
            matched = next(
                (r for r in policy.rules
                 if any(fnmatch.fnmatch(path, g) for g in r.globs)),
                None,
            )
            return PermissionDecision(
                verdict=verdict,
                source=source_name,
                matched_rule=matched,
                reason=matched.reason if matched else "",
            )
    return PermissionDecision(verdict=Verdict.ASK, source="default")
```

---

## SDK Integration — Wrapping Tool Execution

The permission layer sits between the model's tool-use response and your actual
tool dispatch. Both SDKs surface tool calls the same way: you iterate over them,
check permissions, and only execute the ones that pass.

=== "Anthropic"

    ```python
    import anthropic

    client = anthropic.Anthropic()

    def guarded_tool_loop(
        messages: list[dict],
        policy: PermissionPolicy,
        layers: list[tuple[str, PermissionPolicy]],
    ) -> str:
        """Run a tool-use loop with permission checks before each call."""
        while True:
            response = client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                tools=TOOL_DEFINITIONS,
                messages=messages,
            )
            if response.stop_reason != "tool_use":
                return response.content[0].text

            tool_results = []
            for block in response.content:
                if block.type != "tool_use":
                    continue

                # --- permission gate ---
                path = block.input.get("path", "")
                decision = evaluate(layers, path)

                if decision.verdict is Verdict.DENY:
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": f"DENIED: {decision.reason}",
                        "is_error": True,
                    })
                    continue

                if decision.verdict is Verdict.ASK:
                    if not prompt_user(block):  # your UI prompt
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": "User declined.",
                            "is_error": True,
                        })
                        continue

                # --- execute ---
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })

            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
    ```

=== "OpenAI"

    ```python
    from openai import OpenAI

    client = OpenAI()

    def guarded_tool_loop(
        messages: list[dict],
        policy: PermissionPolicy,
        layers: list[tuple[str, PermissionPolicy]],
    ) -> str:
        """Run a tool-use loop with permission checks before each call."""
        while True:
            response = client.chat.completions.create(
                model="gpt-4o",
                tools=TOOL_DEFINITIONS,
                messages=messages,
            )
            choice = response.choices[0]
            if choice.finish_reason != "tool_calls":
                return choice.message.content

            messages.append(choice.message)

            for call in choice.message.tool_calls:
                import json
                args = json.loads(call.function.arguments)

                # --- permission gate ---
                path = args.get("path", "")
                decision = evaluate(layers, path)

                if decision.verdict is Verdict.DENY:
                    messages.append({
                        "role": "tool",
                        "tool_call_id": call.id,
                        "content": f"DENIED: {decision.reason}",
                    })
                    continue

                if decision.verdict is Verdict.ASK:
                    if not prompt_user(call):  # your UI prompt
                        messages.append({
                            "role": "tool",
                            "tool_call_id": call.id,
                            "content": "User declined.",
                        })
                        continue

                # --- execute ---
                result = execute_tool(call.function.name, args)
                messages.append({
                    "role": "tool",
                    "tool_call_id": call.id,
                    "content": result,
                })
    ```

---

## Bash AST Security Overview

Shell commands must never be classified with regex alone. The harness uses
**tree-sitter** to parse commands into an AST and walks the tree looking for
three categories of risk:

| Category | Examples | Why it matters |
|----------|----------|----------------|
| **QuoteContext** | `bash -c "rm -rf /"`, heredocs | Dangerous commands can hide inside string literals that regex misses. |
| **CompoundStructure** | `cmd1 \| cmd2`, `$(subshell)`, `cmd1 && cmd2` | A safe command piped into an unsafe one inherits the risk. |
| **DangerousPatterns** | `rm -rf`, `chmod 777`, `curl \| sh`, `> /dev/sda` | Known destructive idioms that should always flag. |

Tree-sitter gives you a typed node tree (`command_name`, `pipeline`,
`command_substitution`, etc.) so you can write visitors that inspect
structure, not text. For the full classification logic and YOLO-mode
integration, see [Chapter 12 — YOLO Classifier](yolo-classifier.md).

---

## Build-It-Yourself Checklist

- [ ] Define `Verdict` enum with Allow / Deny / Ask
- [ ] Implement `PermissionRule` with glob patterns and a verdict
- [ ] Build `PermissionPolicy` as an immutable, ordered rule set
- [ ] Layer policies by source: policy > project > user > CLI > session
- [ ] Add `check_path` with `fnmatch` glob matching
- [ ] Add `check_command` with risk-score thresholds
- [ ] Wire the permission gate into your tool-use loop (before execution)
- [ ] Return denied/ask results as tool errors so the model can adapt
- [ ] Use tree-sitter (not regex) for shell command parsing
- [ ] Persist session grants so users don't re-approve the same action
- [ ] Log every `PermissionDecision` for auditability

---

*Next: [Chapter 12 — YOLO Classifier](yolo-classifier.md) covers the side-query
classifier that auto-approves or blocks tool calls in auto-mode.*
