# Chapter 7: Multi-Agent Orchestration

Sub-agents are where harnesses stop being single-loop tools and become orchestration systems. This chapter explains spawning, coordinator mode, cache sharing, and routing.

## What this chapter covers

- agent spawning
- coordinator mode
- prompt cache sharing
- send-message routing

## Why it matters

Multi-agent systems create leverage and complexity at the same time. If delegation rules are fuzzy, the product becomes expensive, redundant, and hard to trust.

## Key ideas

- define parent and child responsibilities clearly
- preserve prompt cache stability across forks when possible
- route messages intentionally instead of letting threads drift
- use coordination modes only when they materially improve outcomes

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-7-multi-agent-orchestration)
- Key subsections:
  - `7.1` Agent Spawning
  - `7.2` Coordinator Mode
  - `7.3` Prompt Cache Sharing
  - `7.4` SendMessage Routing
