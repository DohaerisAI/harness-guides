# Chapter 4: Bootstrap And Startup

Bootstrap gets the harness from a cold process to a ready runtime. This chapter covers startup sequencing, deferred initialization, and system prompt assembly.

## What this chapter covers

- the seven-stage startup pipeline
- prompt assembly at bootstrap time
- latency hiding through parallel initialization
- a minimal build-it-yourself recipe

## Why it matters

Startup architecture shapes both first impression and correctness. A harness that initializes the wrong things too early creates trust, performance, and state problems immediately.

## Key ideas

- front-load only what is needed for the first turn
- defer expensive or trust-sensitive work
- treat prompt assembly as runtime infrastructure
- use parallel I/O deliberately

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-4-bootstrap-startup)
- Key subsections:
  - `4.1` The 7-Stage Pipeline
  - `4.2` System Prompt Assembly
  - `4.3` Build It Yourself
