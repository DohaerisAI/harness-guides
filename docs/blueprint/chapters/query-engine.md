# Chapter 3 — Query Engine & Conversation Loop

The query loop is the heartbeat — it calls the model, executes tools, handles errors, and decides when to stop.

## Why this system exists

Every coding agent lives or dies by its conversation loop. This is the single piece of code that transforms a stateless LLM API into an autonomous agent: it sends messages, inspects whether the model wants to call a tool or emit final text, executes tool calls, feeds results back, and repeats until the task is done or a budget gate fires. Get this loop wrong and the agent either halts too early, burns money in infinite cycles, or loses context mid-task.

## The core loop (shared architecture)

Regardless of which provider you use, every query engine follows the same shape:

```text
while True:
    response = call_model(messages, tools)
    if no tool calls in response:
        break                              # model is done
    for each tool_call in response:
        result = execute(tool_call)
        append result to next input
    if budget_exceeded(turns, cost, tokens):
        break                              # safety stop
```

The differences between providers are **structural** (how messages and tool results are formatted) rather than **architectural** (the loop itself is identical).

## Python implementation

=== "Anthropic"

    ```python
    import anthropic


    async def query_loop(
        client: anthropic.Anthropic,
        messages: list[dict],
        system: str,
        tools: list[dict],
        tool_executor,
        max_turns: int = 10,
    ):
        """Run the agentic loop using Anthropic Messages API."""
        response = None

        for turn in range(max_turns):
            response = client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=8096,
                system=system,
                messages=messages,
                tools=tools,
            )

            # Append assistant response
            messages.append({"role": "assistant", "content": response.content})

            # Check if model wants to use tools
            tool_use_blocks = [b for b in response.content if b.type == "tool_use"]
            if not tool_use_blocks:
                break  # Model is done — no more tool calls

            # Execute each tool and collect results
            tool_results = []
            for block in tool_use_blocks:
                result = await tool_executor.run(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result),
                })

            # Append tool results as user message
            messages.append({"role": "user", "content": tool_results})

        # Return final text
        return next(
            (b.text for b in response.content if hasattr(b, "text")),
            "",
        )
    ```

=== "OpenAI"

    ```python
    import json

    from openai import OpenAI


    def query_loop(
        client: OpenAI,
        user_input: str,
        instructions: str,
        tools: list[dict],
        tool_executor,
        max_turns: int = 10,
        previous_response_id: str | None = None,
    ):
        """Run the agentic loop using OpenAI Responses API."""
        current_input = user_input
        response = None

        for turn in range(max_turns):
            response = client.responses.create(
                model="gpt-4.1",
                instructions=instructions,
                input=current_input,
                tools=tools,
                previous_response_id=previous_response_id,
            )
            previous_response_id = response.id

            # Check for function calls
            function_calls = [
                item for item in response.output
                if item.type == "function_call"
            ]
            if not function_calls:
                break  # Model is done

            # Execute and build next input
            results = []
            for call in function_calls:
                result = tool_executor.run(call.name, json.loads(call.arguments))
                results.append({
                    "type": "function_call_output",
                    "call_id": call.call_id,
                    "output": str(result),
                })
            current_input = results

        # Return final text
        return next(
            (item.text for item in response.output if hasattr(item, "text")),
            "",
        )
    ```

!!! note "Conversation state ownership"
    Anthropic requires you to manage the full `messages` list yourself — every assistant turn and tool result must be appended explicitly. OpenAI's Responses API chains turns via `previous_response_id`, so the server holds prior context. Both approaches work; the tradeoff is control vs. convenience.

## Streaming

For interactive agents, streaming lets users see partial output as it arrives instead of waiting for the full response.

=== "Anthropic"

    ```python
    import anthropic


    def stream_response(
        client: anthropic.Anthropic,
        messages: list[dict],
        system: str,
        tools: list[dict],
    ):
        """Stream a single model turn with Anthropic."""
        with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=8096,
            system=system,
            messages=messages,
            tools=tools,
        ) as stream:
            for event in stream:
                match event.type:
                    case "content_block_start":
                        # A new content block (text or tool_use) is starting
                        pass
                    case "content_block_delta":
                        if hasattr(event.delta, "text"):
                            print(event.delta.text, end="", flush=True)
                    case "message_stop":
                        # Full message received — inspect for tool calls
                        pass

        # After stream closes, get the accumulated message
        return stream.get_final_message()
    ```

=== "OpenAI"

    ```python
    from openai import OpenAI


    def stream_response(
        client: OpenAI,
        current_input,
        instructions: str,
        tools: list[dict],
    ):
        """Stream a single model turn with OpenAI."""
        stream = client.responses.create(
            model="gpt-4.1",
            instructions=instructions,
            input=current_input,
            tools=tools,
            stream=True,
        )
        for event in stream:
            if event.type == "response.output_text.delta":
                print(event.delta, end="", flush=True)
    ```

!!! tip "Streaming + tool loops"
    In a full agent, you stream each turn individually, then check the accumulated response for tool calls. The outer `while` loop stays the same — only the inner `call_model` step switches from blocking to streaming.

## Recovery mechanisms

Things go wrong. The query engine is where you catch and recover from failures instead of letting them crash the session.

| Scenario | Detection | Recovery | Limit |
|----------|-----------|----------|-------|
| Prompt too long | API error (context length exceeded) | Compact messages — summarize older turns | 1 attempt |
| Max output tokens | `stop_reason == "max_tokens"` | Truncate partial output, continue in next turn | 3 attempts |
| Rate limit | HTTP 429 | Exponential backoff with jitter | 5 retries |
| Model error | HTTP 500 | Retry with exponential backoff | 3 retries |

```python
import time
import random


def retry_with_backoff(fn, max_retries: int = 3, base_delay: float = 1.0):
    """Retry a callable with exponential backoff and jitter."""
    for attempt in range(max_retries):
        try:
            return fn()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            time.sleep(delay)
```

## Budget gates

Budget gates prevent runaway agents. Check them after every turn.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class BudgetGates:
    max_turns: int = 10
    max_cost_usd: float = 5.0
    max_tokens: int = 200_000


def check_budget(
    gates: BudgetGates,
    turn_count: int,
    total_cost: float,
    total_tokens: int,
) -> str | None:
    """Return a reason string if any budget gate is hit, else None."""
    if turn_count >= gates.max_turns:
        return "max_turns_reached"
    if total_cost >= gates.max_cost_usd:
        return "max_cost_reached"
    if total_tokens >= gates.max_tokens:
        return "max_tokens_reached"
    return None
```

!!! warning "Always enforce budget gates"
    An agent without budget gates is a credit card attached to a `while True` loop. Even during development, set conservative defaults and log when gates fire.

## Transcript persistence

Crash recovery depends on writing state to disk at the right moments. The pattern is asymmetric on purpose — block before the expensive call, fire-and-forget after it:

```text
Write-before-query pattern:
  User message → save transcript [BLOCKING]    ← crash recovery point
  API call starts...
  Assistant response → save transcript [async]  ← non-blocking
  Turn complete → flush [if eager mode]
```

The blocking write before the API call ensures that if the process crashes mid-request, you can resume from the last user message without re-executing tools. The async write after the response is safe because the worst case (crash before write completes) just means you re-call the model for one turn.

## Build it yourself

A checklist for implementing your own query engine:

- [ ] **One central turn loop** — all model calls flow through a single function
- [ ] **Tool result normalization** — convert executor output to the provider's expected format before re-injection
- [ ] **Budget gates on every turn** — check cost, token count, and turn count after each iteration
- [ ] **Persist before calling** — write user messages to disk before the API call, not after
- [ ] **Retry with backoff** — handle 429s and 500s in the loop, not in calling code

## Key differences: Anthropic vs OpenAI

| Concept | Anthropic | OpenAI |
|---------|-----------|--------|
| Conversation state | `messages` list (user/assistant) | `previous_response_id` chain |
| Tool call signal | `stop_reason == "tool_use"` | Output contains `function_call` items |
| Tool result format | `tool_result` content block in user message | `function_call_output` in next input |
| Streaming | `client.messages.stream()` context manager | `stream=True` parameter |
| System prompt | `system` parameter | `instructions` parameter |
| Token counting | `response.usage.input_tokens` / `output_tokens` | `response.usage.input_tokens` / `output_tokens` |
