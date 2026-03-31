# Chapter 3: Query Engine And Conversation Loop

The query engine is the harness control center. It sends model requests, executes tool calls, injects results, persists transcripts, and recovers when turns go sideways.

## What this chapter covers

- the `QueryEngine` class
- the `submitMessage()` entry point
- `queryLoop()` as the central control loop
- recovery mechanisms and budget gates
- transcript persistence strategy

## Why it matters

This chapter is where agent products either become reliable or chaotic. Loop design determines whether tool use, retries, and state updates remain understandable under pressure.

## Key ideas

- keep the main loop legible
- separate mutable conversation history from broader app state
- design for resume and crash recovery early
- make token and cost budgets first-class runtime rules

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-3-query-engine-conversation-loop)
- Key subsections:
  - `3.1` QueryEngine Class
  - `3.2` `submitMessage()` Entry Point
  - `3.3` `queryLoop()`
  - `3.4` Recovery Mechanisms
  - `3.5` Budget Gates
  - `3.6` Transcript Persistence Strategy
