# Chapter 8: MCP Integration

MCP turns the harness into a broader tool and context platform. This chapter covers transports, connection lifecycle, reconnection strategy, and deduplication.

## What this chapter covers

- transport types
- connection lifecycle
- reconnection strategy
- connection states
- plugin MCP deduplication

## Why it matters

External tool servers expand capability quickly, but they also widen the failure surface. The harness needs to model connection state as a first-class concern.

## Key ideas

- treat transport selection as architecture, not plumbing
- model reconnect behavior explicitly
- make connection states visible to the runtime
- deduplicate tool surfaces before they hit the prompt

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-8-mcp-integration)
- Key subsections:
  - `8.1` Transport Types
  - `8.2` Connection Lifecycle
  - `8.3` Reconnection Strategy
  - `8.4` Connection States
  - `8.5` Plugin MCP Dedup
