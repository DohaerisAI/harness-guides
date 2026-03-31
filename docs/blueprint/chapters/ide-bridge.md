# Chapter 11: IDE Bridge Protocol

The IDE bridge lets the harness operate beyond the terminal. This chapter covers the bridge architecture, key types, spawn modes, permission proxying, and backoff behavior.

## What this chapter covers

- bridge architecture
- key protocol types
- spawn modes
- permission proxying
- backoff configuration

## Why it matters

Once an agent spans both terminal and IDE surfaces, protocol clarity matters as much as tool clarity. Weak bridge design produces race conditions and inconsistent trust behavior.

## Key ideas

- make the bridge protocol typed and explicit
- proxy permissions without hiding who made the decision
- define spawn modes clearly
- treat reconnection and backoff as product behavior

## Read the full chapter

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-11-ide-bridge-protocol)
- Key subsections:
  - `11.1` Architecture
  - `11.2` Key Types
  - `11.3` Spawn Modes
  - `11.4` Permission Proxying
  - `11.5` Backoff Configuration
