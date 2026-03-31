# Chapter 12 -- Memory & Cost Tracking

Agents get expensive and forgetful fast. This chapter covers two systems
that decide whether a harness survives past the demo: **persistent memory**
(so the agent remembers you) and **cost tracking** (so you can afford it).

---

## Memory System

### MEMORY.md as an index file

Memory lives in a single Markdown file (`MEMORY.md`) at the project or user
scope. Each entry is one line, under 150 characters, pointing to a topic or
capturing a fact. The file is an *index*, not a knowledge base -- keep it
scannable.

```text title="~/.claude/MEMORY.md"
[user] prefers frozen dataclasses over mutable state
[project] deploy target is fly.io, region ord
[feedback] confirmed: always use ruff, never black
[reference] API docs at https://internal.dev/api/v3
```

### Memory types taxonomy

Every entry carries a type prefix in square brackets. Four types cover the
space:

| Type | Purpose | Example |
|------|---------|---------|
| `user` | Role, goals, preferences | `[user] backend engineer, Python 3.12` |
| `feedback` | Corrections and confirmations | `[feedback] use pytest-asyncio, not anyio` |
| `project` | Ongoing work, deadlines, context | `[project] shipping v2.1 by Friday` |
| `reference` | Pointers to external systems | `[reference] staging DB is on port 5433` |

### Frontmatter schema

When memory entries are stored as structured objects (e.g. in JSON), use
this schema:

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass(frozen=True)
class MemoryEntry:
    """Single memory entry with type classification."""

    type: str          # "user" | "feedback" | "project" | "reference"
    content: str       # The memory text, <150 chars
    source: str = ""   # Which conversation or file produced this

    def to_line(self) -> str:
        return f"[{self.type}] {self.content}"
```

### Dual-cap truncation

MEMORY.md must stay small enough to inject into every prompt. Two caps
enforce this -- **line count first, then byte size**:

```python
MAX_LINES = 200
MAX_BYTES = 25_000


def truncate_memory(raw: str) -> str:
    """Apply dual-cap truncation: 200 lines AND 25 KB.

    Line cap is applied first. If the result still exceeds the byte
    cap, truncate at the last newline boundary within the byte limit.
    """
    lines = raw.strip().split("\n")
    truncated = (
        "\n".join(lines[:MAX_LINES]) if len(lines) > MAX_LINES else raw.strip()
    )

    if len(truncated.encode("utf-8")) > MAX_BYTES:
        cut = truncated.rfind("\n", 0, MAX_BYTES)
        truncated = truncated[: cut if cut > 0 else MAX_BYTES]

    return truncated
```

!!! info "Why two caps?"
    Line count catches the common case (many short entries). Byte cap
    catches the edge case (fewer entries with long content or unicode).
    Applying line-first means the byte cap rarely fires.

### MemoryStore: read, append, search

The full store wraps the file with append, load, and search operations.
Every mutation returns a new string (the file content) rather than
modifying state in place.

```python
from __future__ import annotations

import os
from dataclasses import dataclass


@dataclass(frozen=True)
class MemoryStore:
    """File-backed memory store with dual-cap truncation."""

    path: str

    def load(self) -> str:
        """Read the memory file, applying truncation."""
        if not os.path.exists(self.path):
            return ""
        with open(self.path, encoding="utf-8") as f:
            return truncate_memory(f.read())

    def append(self, entry: MemoryEntry) -> str:
        """Append an entry and return the (possibly truncated) result."""
        current = self.load()
        updated = (
            f"{current}\n{entry.to_line()}" if current else entry.to_line()
        )
        truncated = truncate_memory(updated)
        with open(self.path, "w", encoding="utf-8") as f:
            f.write(truncated)
        return truncated

    def search(self, query: str) -> list[str]:
        """Return lines that contain the query (case-insensitive)."""
        content = self.load()
        query_lower = query.lower()
        return [
            line
            for line in content.split("\n")
            if query_lower in line.lower()
        ]
```

### Auto-memory: detecting when to save

The harness can detect memory-worthy moments from conversation content.
Pattern-match on explicit signals rather than saving everything:

```python
import re

# Patterns that suggest the user is stating a preference or correction
MEMORY_SIGNALS: list[tuple[str, str]] = [
    (r"(?i)\balways\b.*\buse\b", "feedback"),
    (r"(?i)\bnever\b.*\buse\b", "feedback"),
    (r"(?i)\bprefer\b", "user"),
    (r"(?i)\bmy role\b", "user"),
    (r"(?i)\bdeadline\b", "project"),
    (r"(?i)\bdue\b.*\b(monday|tuesday|wednesday|thursday|friday)\b", "project"),
    (r"(?i)\bdocs?\s+(at|are|live)\b", "reference"),
]


def detect_memory_candidates(text: str) -> list[MemoryEntry]:
    """Scan assistant or user text for memory-worthy statements.

    Returns candidate entries. The harness should confirm with the user
    before persisting.
    """
    candidates: list[MemoryEntry] = []
    for pattern, memory_type in MEMORY_SIGNALS:
        if re.search(pattern, text):
            # Extract the sentence containing the match
            for sentence in re.split(r"[.!?\n]", text):
                if re.search(pattern, sentence):
                    content = sentence.strip()[:150]
                    if content:
                        candidates.append(
                            MemoryEntry(type=memory_type, content=content)
                        )
    return candidates
```

!!! warning "Always confirm before saving"
    Auto-detected memories should be presented to the user for approval.
    Silent memory writes erode trust.

---

## Cost Tracking

Cost tracking uses the `CostTracker` and pricing table defined in
[Chapter: Model Pricing](model-pricing.md). This section covers the
**integration patterns** -- how the harness wires cost awareness into the
agent loop.

### Brief reference: CostTracker

The full implementation lives in `model-pricing.md`. The key interface:

```python
from dataclasses import dataclass, field

from model_pricing import calculate_cost


@dataclass
class CostTracker:
    """Accumulates cost across an entire session."""

    total_cost_usd: float = 0.0
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    turn_costs: list[float] = field(default_factory=list)

    def record(self, model: str, usage: dict) -> float:
        """Record one API call. Returns cost of this call."""
        cost = calculate_cost(
            model,
            usage.get("input_tokens", 0),
            usage.get("output_tokens", 0),
            usage.get("cache_creation_input_tokens", 0),
            usage.get("cache_read_input_tokens", 0),
        )
        self.total_cost_usd += cost
        self.total_input_tokens += usage.get("input_tokens", 0)
        self.total_output_tokens += usage.get("output_tokens", 0)
        self.turn_costs.append(cost)
        return cost
```

### Per-turn cost display

Show the user what each turn cost. Transparency builds trust and helps
users develop intuition for what operations are expensive:

```python
def format_turn_cost(turn_cost: float, session_total: float) -> str:
    """Format a single turn's cost for display."""
    return f"Turn: ${turn_cost:.4f} | Session: ${session_total:.4f}"
```

### Budget enforcement at the loop level

Budget checks belong **inside the agent loop**, not in a disconnected
analytics module. Check *before* each API call so you never overshoot:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class BudgetPolicy:
    """Spending limits for a session."""

    max_cost_usd: float = 5.0
    max_tokens: int = 500_000
    warn_at_pct: float = 0.8


@dataclass(frozen=True)
class BudgetStatus:
    """Result of a budget check -- immutable snapshot."""

    exceeded: bool
    warning: bool
    message: str


def check_budget(tracker: CostTracker, policy: BudgetPolicy) -> BudgetStatus:
    """Check whether the session has exceeded or is approaching its budget."""
    total_tokens = tracker.total_input_tokens + tracker.total_output_tokens

    if tracker.total_cost_usd >= policy.max_cost_usd:
        return BudgetStatus(
            exceeded=True,
            warning=False,
            message=f"Budget exceeded: ${tracker.total_cost_usd:.4f} >= ${policy.max_cost_usd}",
        )

    if total_tokens >= policy.max_tokens:
        return BudgetStatus(
            exceeded=True,
            warning=False,
            message=f"Token limit exceeded: {total_tokens:,} >= {policy.max_tokens:,}",
        )

    cost_pct = tracker.total_cost_usd / policy.max_cost_usd
    if cost_pct >= policy.warn_at_pct:
        return BudgetStatus(
            exceeded=False,
            warning=True,
            message=f"Budget warning: ${tracker.total_cost_usd:.4f} ({cost_pct:.0%} of ${policy.max_cost_usd})",
        )

    return BudgetStatus(exceeded=False, warning=False, message="")
```

### Wiring it into the agent loop

The budget check gates each iteration. When the budget is exceeded, the
loop exits gracefully instead of cutting mid-response:

```python
def agent_loop(
    tracker: CostTracker,
    policy: BudgetPolicy,
    memory: MemoryStore,
) -> None:
    """Simplified agent loop showing memory + cost integration."""
    while True:
        user_input = input("> ")
        if user_input.lower() in ("quit", "exit"):
            break

        # Pre-turn budget check
        status = check_budget(tracker, policy)
        if status.exceeded:
            print(f"Stopping: {status.message}")
            break
        if status.warning:
            print(f"Warning: {status.message}")

        # Inject memory into the system prompt
        memory_context = memory.load()

        # ... build messages, call API, get response + usage ...

        # Post-turn cost recording (provider-specific, see tabs below)
        # turn_cost = tracker.record(model, usage_dict)
        # print(format_turn_cost(turn_cost, tracker.total_cost_usd))

        # Check for memory-worthy content in the response
        # candidates = detect_memory_candidates(response_text)
```

### SDK usage extraction

Extracting usage differs between providers. Use provider-specific
adapters to normalize into the `dict` that `CostTracker.record` expects:

=== "Anthropic"

    ```python
    import anthropic


    def extract_anthropic_usage(
        response: anthropic.types.Message,
    ) -> tuple[str, dict]:
        """Extract model name and usage dict from an Anthropic response."""
        return response.model, {
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "cache_read_input_tokens": getattr(
                response.usage, "cache_read_input_tokens", 0
            ),
            "cache_creation_input_tokens": getattr(
                response.usage, "cache_creation_input_tokens", 0
            ),
        }


    # Usage in the loop:
    # response = client.messages.create(model="claude-sonnet-4-6", ...)
    # model, usage = extract_anthropic_usage(response)
    # turn_cost = tracker.record(model, usage)
    ```

=== "OpenAI"

    ```python
    import openai


    def extract_openai_usage(
        response: openai.types.responses.Response,
    ) -> tuple[str, dict]:
        """Extract model name and usage dict from an OpenAI response."""
        return response.model, {
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            # OpenAI does not expose cache token fields in the same way
        }


    # Usage in the loop:
    # response = client.responses.create(model="gpt-4.1", ...)
    # model, usage = extract_openai_usage(response)
    # turn_cost = tracker.record(model, usage)
    ```

---

## Putting it together

The memory system and cost tracker converge in the session loop. Memory
context is injected into every prompt; cost is checked before every API
call. Both are inspectable -- the user can read `MEMORY.md` directly, and
the cost tracker exposes `turn_costs` for full auditability.

!!! tip "Build checklist"
    1. Store memory in a plain-text file with typed prefixes
    2. Enforce dual-cap truncation (200 lines, 25 KB)
    3. Confirm auto-detected memories before saving
    4. Track cost per-turn using the pricing table from [model-pricing](model-pricing.md)
    5. Check budget *before* each API call, not after
    6. Show per-turn cost to the user for transparency
