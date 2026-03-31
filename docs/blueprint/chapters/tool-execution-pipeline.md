# Chapter 2: Tool Execution Pipeline

This chapter explains the path from a model-emitted tool call to actual execution. It focuses on the six-phase pipeline, async orchestration, streaming execution, and progress reporting.

## What this chapter covers

- the six-phase execution pipeline
- `runToolUse()` as the core async generator
- `StreamingToolExecutor` and concurrency rules
- progress callback patterns for long-running tools

## Why it matters

Most runtime risk sits here. The model asks for a tool, but the harness decides whether it runs now, later, in parallel, or not at all.

## Key ideas

- treat execution as a pipeline, not a single function call
- make safe tools parallel only when explicitly classified that way
- stream progress for long-running tools so users are not blind
- keep permission, validation, and hooks in a stable order

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-2-tool-execution-pipeline)
- Key subsections:
  - `2.1` The 6-Phase Pipeline
  - `2.2` `runToolUse()`
  - `2.3` `StreamingToolExecutor`
  - `2.4` Progress Callbacks
  - `2.5` Build It Yourself
