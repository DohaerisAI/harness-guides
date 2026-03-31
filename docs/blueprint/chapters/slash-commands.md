# Chapter 9: Slash Command System

Slash commands are the human-facing command grammar of the harness. In Python, they are often the simplest place to keep structured user intent out of freeform chat.

## Why this system exists

Once users rely on repeatable workflows, they need more than conversational luck. Slash commands give the harness a discoverable, structured entry surface.

## Shared architecture

- commands map user intent to runtime actions
- registry and discovery should be centralized
- command execution should pass through the same safety and state boundaries as chat-originated work

## Python implementation

Use one registry that can dispatch:

- local built-ins like `/help`
- workflow commands like `/resume`
- higher-level automation commands that still enter the query loop or tool executor

Keep command handlers small and explicit. They should call runtime systems, not replace them.

## OpenAI Responses API mapping

Commands are usually a runtime feature, not a Responses API feature. But they often end in one of two actions:

- mutate local runtime state
- call the model through `responses.create`

That means command outputs should already be shaped for the same loop and transcript system used by freeform requests.

## Failure modes and tradeoffs

- command logic bypasses core runtime state
- hidden commands with no discoverability
- command handlers that duplicate query-loop logic
- commands becoming a second permission model

## Build-it-yourself checklist

- define one command registry
- keep handlers small
- route command effects through existing runtime systems
- make command discovery easy
- avoid a second parallel architecture just for commands

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-9-slash-command-system)
- the source discovery here centered on command registry patterns and skill loading
