# Chapter 10: Terminal UI

This chapter explains the terminal UI from a Python builder's perspective: the operator needs visibility into the loop, not a pretty disguise over runtime confusion.

## Why this system exists

Terminal UI is part of the trust model. If users cannot see what the harness is doing, they will assume the model is smarter or dumber than it really is.

## Shared architecture

- render the current turn state
- show tool progress and permission prompts clearly
- preserve transcript readability
- avoid burying important state in decorative output

## Python implementation

The UI stack can vary, but the runtime contract should not. The interface layer should consume structured events like:

- `model_started`
- `tool_requested`
- `tool_progress`
- `permission_required`
- `turn_completed`

That keeps the UI from reaching into business logic to guess what happened.

## OpenAI Responses API mapping

Responses API streaming becomes especially useful here. If you use `stream=True`, the UI can render incremental output and tool activity without pretending that the whole turn is a single blocking blob.

The runtime should translate API events into UI events instead of exposing raw transport details directly.

## Failure modes and tradeoffs

- UI code owns business state
- tool progress is invisible
- permission prompts look the same as ordinary output
- raw API events leak directly into presentation logic

## Build-it-yourself checklist

- define structured runtime events
- render progress and permission prompts distinctly
- keep the transcript readable under long tool runs
- use streaming only when the UI can display it coherently
- keep the UI downstream of runtime state

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-10-terminal-ui-reactink)
- the original discovery material here came from the terminal component hierarchy and per-tool rendering model
