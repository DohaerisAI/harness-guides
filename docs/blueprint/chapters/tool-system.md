# Chapter 1: Tool System

Tools are the harness surface that turns a model into an actor. This chapter defines the core tool contract, safe defaults, registration flow, and a reference implementation pattern.

## What this chapter covers

- the `Tool` interface as the system contract
- why `buildTool()` uses fail-closed defaults
- how built-in and MCP tools are assembled into one prompt-stable pool
- a concrete `GlobTool` example
- a minimal recipe for building your own tool system

## Why it matters

If the tool contract is vague, everything above it gets worse: permissions become ad hoc, execution order becomes unclear, and prompt behavior drifts. Strong harnesses make tool boundaries explicit very early.

## Key ideas

- define validation and permission checks separately
- default unknown tools toward safer execution assumptions
- keep tool registration declarative
- stabilize tool ordering so prompt cache behavior stays predictable

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-1-tool-system)
- Key subsections:
  - `1.1` The Tool Interface
  - `1.2` `buildTool()` Factory
  - `1.3` Tool Registration
  - `1.4` Reference Implementation: GlobTool
  - `1.5` Build It Yourself
