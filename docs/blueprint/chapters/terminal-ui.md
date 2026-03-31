# Chapter 10: Terminal UI

This chapter explains why a React/Ink terminal UI can make sense for an agent harness and how the rendering tree maps to runtime events and per-tool output.

## What this chapter covers

- why React in a terminal
- component hierarchy
- per-tool rendering
- implementation guidance

## Why it matters

Terminal UIs are not just presentation. They shape how trust, progress, permission prompts, and tool results are perceived by the operator.

## Key ideas

- tie rendering structure to runtime structure
- make tool output legible without burying the main task
- use component boundaries to localize UI complexity
- optimize for operator clarity, not novelty

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-10-terminal-ui-reactink)
- Key subsections:
  - `10.1` Why React in a Terminal
  - `10.2` Component Hierarchy
  - `10.3` Per-Tool Rendering
  - `10.4` Build It Yourself
