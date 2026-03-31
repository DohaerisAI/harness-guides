# Chapter 8: MCP Integration

MCP turns the harness into a broader tool and context platform. For Python builders, this chapter is about treating remote capability as part of the runtime model instead of a magical side channel.

## Why this system exists

MCP expands what the model can reach, but it also expands failure surface. The harness needs a clean integration layer before it exposes remote tools to the model.

## Shared architecture

- treat MCP connections as runtime-managed resources
- maintain connection state explicitly
- deduplicate tools before exposing them
- handle reconnect and degraded mode intentionally

## Python implementation

Build one adapter layer that converts MCP-discovered tools into your internal tool registry format. That keeps the rest of the harness from caring whether a capability is local or remote.

Keep per-server state such as:

- transport type
- connection status
- last heartbeat or error
- exported tools

## OpenAI Responses API mapping

OpenAI's tools guidance and Responses surface make MCP especially relevant because the model can reason over a single exposed tool set while the runtime decides which capabilities are local and which are remote.

The harness should still own:

- connection setup
- tool deduplication
- permission filtering
- error handling when a remote tool is unavailable

## Failure modes and tradeoffs

- treating remote tools as always available
- exposing duplicate tool names
- no degraded behavior when an MCP server disconnects
- coupling model-facing schemas directly to transport details

## Build-it-yourself checklist

- write an MCP adapter into the internal registry
- track per-server connection state
- deduplicate before exposing tools
- keep local permission policy in front of remote execution
- define degraded behavior for server failure

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-8-mcp-integration)
- the original discovery work here came from transport, lifecycle, reconnect, and dedup behavior
