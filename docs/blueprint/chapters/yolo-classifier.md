# YOLO Classifier — Auto-Mode Permission System

## Why this exists

When users enable "auto mode" (aka YOLO mode), the harness needs to decide which tool calls to auto-approve and which to block — without asking the user every time. This is done by a **side-query classifier**: a separate, fast API call that evaluates the tool action against customizable rules.

The classifier is one of the most interesting patterns from the Claude Code architecture because it solves a real tension: users want speed (no permission prompts), but the harness must prevent destructive actions (deleting files, force-pushing, dropping tables). The YOLO classifier resolves this by offloading the decision to a cheap, fast model that applies user-defined rules.

## Architecture overview

```
Tool call requested
    |
Is tool in safe-allowlist? --- YES ---> auto-approve (skip classifier)
    | NO
Is tool a file edit in CWD? --- YES ---> auto-approve (acceptEdits fast path)
    | NO
Side-query to fast model (Haiku) with:
  - Tool name + input summary
  - Conversation transcript (compressed)
  - Customizable allow/deny rules
    |
Classifier returns: { should_block: bool, reason: str }
    |
should_block=true --- DENY with reason
should_block=false --- ALLOW
```

Three layers of checks run in order. Most tool calls resolve at the first two layers and never hit the classifier at all.

## Safe-allowlist (skip classifier entirely)

These tools are so safe they never need classifier checking. They are read-only, metadata-only, or internal coordination tools:

```python
SAFE_ALLOWLISTED_TOOLS = {
    # Read-only file operations
    "read_file", "grep", "glob", "lsp",

    # Search / discovery
    "tool_search", "list_mcp_resources", "read_mcp_resource",

    # Task management (metadata only)
    "todo_write", "task_create", "task_get",
    "task_update", "task_list", "task_stop",

    # Plan mode / UI interaction
    "ask_user_question", "enter_plan_mode", "exit_plan_mode",

    # Agent coordination (internal mailbox only)
    "team_create", "team_delete", "send_message",

    # Misc safe
    "sleep",
}


def is_auto_mode_allowlisted(tool_name: str) -> bool:
    return tool_name in SAFE_ALLOWLISTED_TOOLS
```

The allowlist is intentionally conservative. Adding a tool here means it can **never** be blocked, even if the user has a deny rule that matches it.

## Customizable rules

Users can customize what auto-mode allows and denies through three rule categories:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class AutoModeRules:
    allow: tuple[str, ...] = ()        # "git status", "npm test", "python -m pytest"
    soft_deny: tuple[str, ...] = ()    # "rm -rf", "DROP TABLE", "force push"
    environment: tuple[str, ...] = ()  # "Node.js project", "uses PostgreSQL"
```

- **allow**: Commands and patterns that should be auto-approved. These are things the user trusts and runs frequently.
- **soft_deny**: Commands and patterns that should be blocked even in auto mode. These are destructive or irreversible operations.
- **environment**: Context about the project that helps the classifier make better decisions. Not rules — just hints.

Rules are passed directly into the classifier prompt, so they can be natural language. A rule like `"never run migrations without asking"` works just as well as `"DROP TABLE"`.

## The classifier side-query

The classifier runs as a **separate API call** using a fast model (Haiku). It receives the tool action plus the rules and returns a structured decision via forced tool use.

=== "Anthropic"

    ```python
    import anthropic
    import json

    async def classify_tool_action(
        client: anthropic.Anthropic,
        tool_name: str,
        tool_input: dict,
        transcript: list[dict],
        rules: AutoModeRules,
    ) -> tuple[bool, str]:
        """Side-query classifier. Returns (should_block, reason)."""

        # Build compressed transcript (last N entries as JSONL)
        transcript_text = "\n".join(
            json.dumps({
                "role": entry["role"],
                "action": entry.get("action", entry.get("content", "")[:200]),
            })
            for entry in transcript[-20:]
        )

        system_prompt = f"""You are a security classifier for an AI coding assistant.
    Evaluate whether this tool action should be ALLOWED or BLOCKED.

    ALLOW rules (auto-approve these):
    {chr(10).join(f'- {r}' for r in rules.allow)}

    DENY rules (block these):
    {chr(10).join(f'- {r}' for r in rules.soft_deny)}

    Environment context:
    {chr(10).join(f'- {r}' for r in rules.environment)}

    Respond with the classify_result tool."""

        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=256,
            system=system_prompt,
            messages=[{
                "role": "user",
                "content": (
                    f"Tool: {tool_name}\n"
                    f"Input: {json.dumps(tool_input)}\n\n"
                    f"Recent transcript:\n{transcript_text}"
                ),
            }],
            tools=[{
                "name": "classify_result",
                "description": "Return classification decision",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "thinking": {
                            "type": "string",
                            "description": "Brief reasoning about the action",
                        },
                        "should_block": {"type": "boolean"},
                        "reason": {
                            "type": "string",
                            "description": "Why allow or block",
                        },
                    },
                    "required": ["thinking", "should_block", "reason"],
                },
            }],
            tool_choice={"type": "tool", "name": "classify_result"},
        )

        for block in response.content:
            if block.type == "tool_use":
                return block.input["should_block"], block.input["reason"]

        return True, "Classifier failed to return decision"
    ```

=== "OpenAI"

    ```python
    from openai import OpenAI
    import json

    async def classify_tool_action(
        client: OpenAI,
        tool_name: str,
        tool_input: dict,
        transcript: list[dict],
        rules: AutoModeRules,
    ) -> tuple[bool, str]:
        """Side-query classifier. Returns (should_block, reason)."""

        transcript_text = "\n".join(
            json.dumps({
                "role": e["role"],
                "action": e.get("action", e.get("content", "")[:200]),
            })
            for e in transcript[-20:]
        )

        response = client.responses.create(
            model="gpt-4.1-mini",
            instructions=f"""You are a security classifier for an AI coding assistant.
    Evaluate whether this tool action should be ALLOWED or BLOCKED.

    ALLOW rules: {json.dumps(rules.allow)}
    DENY rules: {json.dumps(rules.soft_deny)}
    Environment: {json.dumps(rules.environment)}""",
            input=(
                f"Tool: {tool_name}\n"
                f"Input: {json.dumps(tool_input)}\n\n"
                f"Recent transcript:\n{transcript_text}"
            ),
            tools=[{
                "type": "function",
                "name": "classify_result",
                "description": "Return classification decision",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "thinking": {"type": "string"},
                        "should_block": {"type": "boolean"},
                        "reason": {"type": "string"},
                    },
                    "required": ["thinking", "should_block", "reason"],
                },
            }],
            tool_choice={"type": "function", "name": "classify_result"},
        )

        for item in response.output:
            if item.type == "function_call":
                result = json.loads(item.arguments)
                return result["should_block"], result["reason"]

        return True, "Classifier failed"
    ```

Key design decisions in the classifier:

- **Forced tool use** (`tool_choice`) guarantees structured output. No parsing needed.
- **Compressed transcript** gives the classifier conversational context without blowing up token usage. Only the last 20 entries, truncated to 200 chars each.
- **The `thinking` field** forces the model to reason before deciding. This improves accuracy on ambiguous cases.
- **Fail-closed**: if the classifier errors or returns nothing, the action is blocked.

## Full auto-mode permission flow

```python
from enum import Enum


class PermissionDecision(Enum):
    ALLOW = "allow"
    DENY = "deny"
    ASK = "ask"


async def auto_mode_check(
    tool_name: str,
    tool_input: dict,
    context,
    classifier_client,
    rules: AutoModeRules,
) -> tuple[PermissionDecision, str]:
    """Complete auto-mode permission check."""

    # Fast path 1: safe-allowlisted tools
    if is_auto_mode_allowlisted(tool_name):
        return PermissionDecision.ALLOW, "Safe-allowlisted tool"

    # Fast path 2: file edits in working directory (acceptEdits mode)
    if tool_name in ("file_edit", "file_write"):
        file_path = tool_input.get("file_path", "")
        if file_path.startswith(context.working_directory):
            return PermissionDecision.ALLOW, "File edit in working directory"

    # Slow path: side-query classifier
    should_block, reason = await classify_tool_action(
        classifier_client,
        tool_name,
        tool_input,
        context.transcript,
        rules,
    )

    if should_block:
        return PermissionDecision.DENY, reason
    return PermissionDecision.ALLOW, reason
```

The three-layer design means the classifier is only called for genuinely ambiguous actions. In practice, the safe-allowlist and acceptEdits fast paths handle 80%+ of tool calls, so the classifier only fires for bash commands, network requests, and other side-effecting operations.

## Build it yourself

1. **Define a safe-allowlist** of read-only tools that never need classification
2. **Add an acceptEdits fast path** for file operations in the working directory
3. **Implement a side-query classifier** using a fast model with structured output (forced tool use)
4. **Make rules customizable** with allow/deny patterns plus environment context
5. **Log classifier decisions** for debugging — dump the full request/response when a flag is set
6. **Fail closed** — if the classifier errors, block the action and surface the error to the user

!!! tip "Cost optimization"
    The classifier uses a fast, cheap model (Haiku / GPT-4.1-mini). Each classification costs roughly $0.001. The safe-allowlist and acceptEdits fast paths avoid the classifier call for the majority of tool uses, keeping the per-session cost negligible.

!!! warning "Security note"
    The `soft_deny` rules are advisory, not a security boundary. A determined model can rephrase commands to bypass pattern-based deny rules. For hard security boundaries, use the permission system described in the [Permissions chapter](permissions.md) instead.
