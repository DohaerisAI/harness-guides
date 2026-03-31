# Chapter 12: Memory And Cost Tracking

This chapter covers long-lived memory and token economics, two areas that decide whether a harness remains useful after the demo.

## What this chapter covers

- memory systems such as `MEMORY.md`
- token and USD cost tracking
- practical implementation guidance

## Why it matters

Agents become expensive and forgetful quickly if memory and spending are not first-class runtime concerns. The harness needs to expose both clearly.

## Key ideas

- treat memory as managed context, not random notes
- expose cost accounting in a way operators can act on
- keep budgeting connected to loop control
- document the tradeoffs between recall and spend

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-12-memory-cost-tracking)
- Key subsections:
  - `12.1` Memory System
  - `12.2` Cost Tracking
  - `12.3` Build It Yourself
