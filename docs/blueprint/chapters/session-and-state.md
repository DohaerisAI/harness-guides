# Chapter 6: Session And State Management

This chapter focuses on how the harness stores runtime state, compresses context, and persists transcripts without losing the ability to resume or reason about prior turns.

## What this chapter covers

- immutable application state
- React context integration
- context compression
- transcript persistence
- a minimal Python state model

## Why it matters

State design determines whether an agent session is debuggable, resumable, and affordable. Without disciplined state boundaries, every feature leaks into every other feature.

## Key ideas

- keep app state immutable where practical
- make transcript persistence an explicit system
- compress context without destroying user intent
- preserve the difference between runtime settings and conversation history

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-6-session-state-management)
- Key subsections:
  - `6.1` AppState
  - `6.2` React Context Integration
  - `6.3` Context Compression
  - `6.4` Transcript Persistence
  - `6.5` Build It Yourself
