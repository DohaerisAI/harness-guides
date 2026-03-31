# What Is An Agent Harness?

An agent harness is the runtime layer around a model.

The model generates tokens, but the harness is what makes the system actually usable. It decides how tools are exposed, how permissions are checked, how transcripts are stored, how retries work, how state evolves, and how the loop continues after tool calls.

Without the harness, you do not have a serious agent product. You have a model call with a prompt.

## Core responsibilities

- expose tools with stable interfaces
- validate and authorize tool use
- stream model output and tool progress
- persist conversation state and sessions
- enforce cost, safety, and execution limits
- recover from partial failures
- connect the model to local runtime, MCP servers, or IDE bridges

## Why harness quality matters

Most failures in agent products are not caused by the base model alone. They happen in the runtime layer:

- tools run when they should have been blocked
- permissions are confusing or inconsistent
- transcripts get corrupted or lost
- retries duplicate work
- context becomes too large and collapses
- background agents drift from the parent task

Good harnesses make models feel sharper because the environment is coherent. Bad harnesses make strong models look unreliable.

## What to optimize for

- predictable execution
- safe defaults
- traceable behavior
- low operator surprise
- good docs for contributors
- architecture that survives scale

Harness engineering sits between AI UX, systems design, and developer tooling. That is why it deserves its own documentation layer.
