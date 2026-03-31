# Python Path

This is the shortest useful route through Harness Guides if you are trying to build an agent harness in Python.

## What this path optimizes for

- Python-first interfaces and examples
- OpenAI Responses API as the concrete model and tool layer
- architecture that stays portable even if the API provider changes later

## Recommended order

1. [What Is An Agent Harness?](what-is-an-agent-harness.md)
2. [OpenAI Responses API](../blueprint/openai-responses-api.md)
3. [Chapter 1: Tool System](../blueprint/chapters/tool-system.md)
4. [Chapter 3: Query Engine](../blueprint/chapters/query-engine.md)
5. [Chapter 5: Permissions](../blueprint/chapters/permissions.md)
6. [Chapter 6: Session and State](../blueprint/chapters/session-and-state.md)
7. [Chapter 8: MCP Integration](../blueprint/chapters/mcp-integration.md)

## What you should have by the end

- a clear tool interface
- a model loop that can call tools and continue
- a permission layer that is separate from validation
- a transcript and state model
- a path to streaming, MCP, and long-running work

## Working rule

Do not copy surface details from any one product. Copy the harness patterns that remain useful when the UI, model, or provider changes.
