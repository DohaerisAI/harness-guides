# Chapter 3: Query Engine And Conversation Loop

The query engine is the harness control center. In a Python-first harness, it is the explicit loop that calls the model, executes tool work, persists state, and keeps going until the turn is complete.

## Why this system exists

Without a clear loop, the system becomes a pile of one-off API calls and side effects. The query engine is the one place where the runtime remains understandable.

## Shared architecture

- accept user input
- append it to transcript state
- call the model
- inspect tool requests or final output
- execute tools if needed
- continue with tool results
- persist everything needed for resume and audit

## Python implementation

Make the loop boring and explicit.

```python
def run_turn(client, session):
    return client.responses.create(
        model=session.model,
        instructions=session.instructions,
        input=session.pending_input,
        previous_response_id=session.previous_response_id,
        tools=session.tool_defs,
        parallel_tool_calls=True,
    )
```

The important part is not the first request. It is the repeated structure around it:

- persist user input before expensive calls
- keep `previous_response_id` and transcript state together
- store tool results in a normalized internal format
- gate retries and recovery in one place

## OpenAI Responses API mapping

This chapter maps most directly onto:

- `responses.create`
- `previous_response_id`
- `tools`
- `parallel_tool_calls`
- `stream`
- `background`

The loop should treat Responses API as the model surface, not as the source of application state. Your session and transcript store still own the local truth.

## Failure modes and tradeoffs

- mixing transcript state with transient UI state
- not persisting user messages before model calls
- retry logic spread across multiple layers
- no clear boundary between model loop and tool executor

## Build-it-yourself checklist

- write one central turn loop
- store `previous_response_id`
- persist user messages before calling the model
- normalize tool outputs before reinjection
- enforce cost and retry gates centrally

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-3-query-engine-conversation-loop)
- the source discovery here came from the query loop, recovery mechanisms, and transcript persistence logic
