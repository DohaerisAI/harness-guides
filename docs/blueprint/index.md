# Blueprint Overview

The blueprint is organized around the major systems inside an agent CLI, but the current teaching path is intentionally Python-first.

1. Tool system
2. Tool execution pipeline
3. Query engine and conversation loop
4. Bootstrap and startup
5. Permission system
6. Session and state management
7. Multi-agent orchestration
8. MCP integration
9. Slash command system
10. Terminal UI
11. IDE bridge protocol
12. Memory and cost tracking

## Python-first route

- [OpenAI Responses API](openai-responses-api.md)
- [Chapter 1: Tool System](chapters/tool-system.md)
- [Chapter 3: Query Engine](chapters/query-engine.md)
- [Chapter 5: Permissions](chapters/permissions.md)
- [Chapter 6: Session and State](chapters/session-and-state.md)
- [Chapter 8: MCP Integration](chapters/mcp-integration.md)

## Reading tracks

### Core runtime

- [Chapter 1: Tool System](chapters/tool-system.md)
- [Chapter 2: Tool Execution Pipeline](chapters/tool-execution-pipeline.md)
- [Chapter 3: Query Engine](chapters/query-engine.md)
- [Chapter 5: Permissions](chapters/permissions.md)
- [Chapter 6: Session and State](chapters/session-and-state.md)

### Expansion layers

- [Chapter 7: Multi-Agent Orchestration](chapters/multi-agent-orchestration.md)
- [Chapter 8: MCP Integration](chapters/mcp-integration.md)
- [Chapter 9: Slash Commands](chapters/slash-commands.md)
- [Chapter 11: IDE Bridge](chapters/ide-bridge.md)

### Advanced Patterns

- [Streaming Formats](chapters/streaming-formats.md) -- Wire formats and streaming protocols for real-time token delivery
- [YOLO Classifier](chapters/yolo-classifier.md) -- Automatic tool-approval classification and trust scoring
- [Model Pricing](chapters/model-pricing.md) -- Token cost tracking, model selection, and budget-aware routing
- [Hidden Modes](chapters/hidden-modes.md) -- Undocumented operational modes and internal feature flags

### Operator experience

- [Chapter 4: Bootstrap](chapters/bootstrap.md)
- [Chapter 10: Terminal UI](chapters/terminal-ui.md)
- [Chapter 12: Memory and Cost](chapters/memory-and-cost.md)
- [Appendix](chapters/appendix.md)

## Chapter navigation

- [Chapter 1: Tool System](chapters/tool-system.md)
- [Chapter 2: Tool Execution Pipeline](chapters/tool-execution-pipeline.md)
- [Chapter 3: Query Engine](chapters/query-engine.md)
- [Chapter 4: Bootstrap](chapters/bootstrap.md)
- [Chapter 5: Permissions](chapters/permissions.md)
- [Chapter 6: Session and State](chapters/session-and-state.md)
- [Chapter 7: Multi-Agent Orchestration](chapters/multi-agent-orchestration.md)
- [Chapter 8: MCP Integration](chapters/mcp-integration.md)
- [Chapter 9: Slash Commands](chapters/slash-commands.md)
- [Chapter 10: Terminal UI](chapters/terminal-ui.md)
- [Chapter 11: IDE Bridge](chapters/ide-bridge.md)
- [Chapter 12: Memory and Cost](chapters/memory-and-cost.md)
- [Source Provenance](reference-provenance.md)
- [Appendix](chapters/appendix.md)

## Use this section when

- you want the full architecture in one place
- you need a chapter-by-chapter reference
- you are mapping blueprint ideas to your own runtime

The main document still lives at [Full Blueprint](full-blueprint.md), but the chapter pages are now the recommended reading path.

## How to use this section

- Use the chapter pages when you want Python-first implementation guidance.
- Use the full blueprint when you want the exact long-form walkthrough and source-level provenance.
- Use the build-better pages when you want to convert observations into design rules for your own harness.
