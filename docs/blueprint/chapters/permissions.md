# Chapter 5: Permission System

Permissions are what keep a harness from becoming a vague yes-machine. This chapter breaks down context objects, decision chains, classifier support, and shell security boundaries.

## What this chapter covers

- `ToolPermissionContext`
- permission decision order
- auto-mode classification
- bash security controls
- a minimal Python permission model

## Why it matters

Tooling quality is inseparable from safety quality. If authorization is bolted on late, every tool becomes a potential footgun.

## Key ideas

- keep policy layers composable and inspectable
- distinguish allow, deny, and ask behavior clearly
- move dangerous command analysis out of prompt-only logic
- freeze effective permission context for reproducibility

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-5-permission-system)
- Key subsections:
  - `5.1` ToolPermissionContext
  - `5.2` Permission Decision Chain
  - `5.3` Auto-Mode Classifier
  - `5.4` Bash Security
  - `5.5` Minimal Python Model
