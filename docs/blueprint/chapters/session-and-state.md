# Chapter 6: Session And State Management

This chapter focuses on how a Python harness stores runtime state, compresses context, and persists transcripts without losing the ability to resume or reason about prior turns.

## Why this system exists

State design is what determines whether a session is debuggable or mysterious. If transcript state, settings, and temporary UI state collapse into one structure, the harness becomes hard to trust.

## Shared architecture

- transcript state is not the same as UI state
- runtime configuration is not the same as turn-local scratch state
- compression exists to preserve usefulness under context limits
- persistence should favor recoverability over cleverness

## Python implementation

Keep a small set of state objects with clear ownership:

- `SessionConfig`
- `TranscriptStore`
- `TurnState`
- `CostState`

Persist transcript events in append-friendly form and derive UI views from them later. Do not make the UI object the source of truth for the session.

## OpenAI Responses API mapping

Your local session store should track the model-side continuity data, especially `previous_response_id`, but it should not depend on OpenAI storing everything you need for resume and audit.

The runtime should own:

- local transcript records
- local permission outcomes
- local cost accounting
- local compacted summaries if you use them

## Failure modes and tradeoffs

- mixing transport response objects directly into app state
- no persistent turn history
- context compression that drops user intent
- no boundary between settings and transcript data

## Build-it-yourself checklist

- define explicit state objects
- persist transcript records locally
- store `previous_response_id` alongside local session data
- keep compaction explicit and reversible where possible
- make resume a first-class behavior

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-6-session-state-management)
- the original discovery material here came from app state, compaction, and transcript persistence
