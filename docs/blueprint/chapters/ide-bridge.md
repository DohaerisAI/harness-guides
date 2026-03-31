# Chapter 11: IDE Bridge Protocol

The IDE bridge lets the harness operate beyond the terminal. In Python terms, this chapter is about keeping a protocol boundary clean when the harness is no longer the only process in the room.

## Why this system exists

Once the harness is speaking to an IDE or editor companion, protocol clarity matters as much as tool clarity. Otherwise permissions, state, and process ownership become muddy fast.

## Shared architecture

- define typed request/response messages
- separate spawn control from ongoing session messages
- preserve permission boundaries across the bridge
- define reconnect and degraded behavior up front

## Python implementation

Use a small protocol model rather than ad hoc JSON blobs everywhere. Even if the transport is simple, the message taxonomy should be explicit:

- spawn session
- send user input
- request permission
- send tool or progress event
- close session

Keep the bridge thin. It should move structured runtime messages, not reinvent the harness.

## OpenAI Responses API mapping

The bridge does not replace Responses API. It carries harness events and user actions between surfaces while the Python runtime continues to own model calls, tool execution, and permission policy.

## Failure modes and tradeoffs

- protocol types are implicit
- permissions look local when they were really bridged
- reconnect behavior is undefined
- the IDE side becomes a second runtime instead of a client surface

## Build-it-yourself checklist

- define typed bridge messages
- separate spawn from turn traffic
- preserve permission provenance
- define reconnect and shutdown behavior
- keep the bridge thin and explicit

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-11-ide-bridge-protocol)
- the source discovery here came from bridge message types, spawn modes, permission proxying, and backoff rules
