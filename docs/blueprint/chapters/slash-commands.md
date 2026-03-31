# Chapter 9: Slash Command System

Slash commands are the human-facing command grammar of the harness. This chapter covers command types, the registry, and skill loading.

## What this chapter covers

- command types
- command registry
- skill loading
- minimal implementation guidance

## Why it matters

Commands are often the first structured interface users touch. A clean command system turns the harness into a coherent tool instead of a chat box with accidental conventions.

## Key ideas

- keep command registration centralized
- separate built-in commands from dynamic skills cleanly
- make command discovery easy for users
- ensure command execution integrates with the same runtime safety model

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-9-slash-command-system)
- Key subsections:
  - `9.1` Command Types
  - `9.2` Command Registry
  - `9.3` Skill Loading
  - `9.4` Build It Yourself
