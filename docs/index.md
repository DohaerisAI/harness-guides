# Harness Guides

Build better AI agent harnesses.

<div class="hg-hero" markdown>

<p class="hg-kicker">Runtime architecture for serious agent builders</p>

<h2>Turn harness reverse-engineering into docs people actually read.</h2>

<p class="hg-lede">Harness Guides is a public docs site for engineers studying agent runtimes, tool execution, permissions, session management, and multi-agent orchestration. The point is to extract durable patterns, explain them cleanly, and help people build better systems faster.</p>

<div class="hg-signal-row" markdown>

<span class="hg-signal">12 architecture chapters</span>
<span class="hg-signal">docs-native dark mode</span>
<span class="hg-signal">free GitHub Pages deploy</span>

</div>

<div class="hero-actions" markdown>

[Read the Blueprint](blueprint/full-blueprint.md){ .md-button .md-button--primary }
[Browse Chapters](blueprint/index.md){ .md-button }
[Start Here](start-here/what-is-an-agent-harness.md){ .md-button }

</div>

</div>

<div class="hg-grid hg-grid-3" markdown>

<div class="hg-panel" markdown>
### 12 core systems
From tools and permissions to MCP, bridge protocols, memory, and cost tracking.
</div>

<div class="hg-panel" markdown>
### Dark-mode docs
Built on Material for MkDocs with GitHub Pages deployment and a docs-native UX.
</div>

<div class="hg-panel" markdown>
### Builder-first
The site is designed to help people ship harnesses, not just admire internals.
</div>

</div>

## What every serious harness needs

<div class="hg-grid hg-grid-2" markdown>

<div class="hg-panel" markdown>
### Runtime contracts
Tool schemas, validation, permissions, concurrency rules, result formatting, and deterministic registration. These are the boundaries that keep the system legible.
</div>

<div class="hg-panel" markdown>
### Control loops
Conversation state, retries, recovery, transcript persistence, context compression, and budget gates. This is the difference between a demo and an actual operator tool.
</div>

<div class="hg-panel" markdown>
### Safety systems
Trust gates, permission chains, shell constraints, and explicit state transitions. Agents fail hardest where the runtime is vague.
</div>

<div class="hg-panel" markdown>
### Expansion surfaces
MCP, sub-agents, slash commands, and IDE bridges. Capability grows fast here, but so does complexity, so the docs have to keep up.
</div>

</div>

## Why this site exists

- Turn dense reverse-engineering notes into readable public documentation.
- Give developers a common vocabulary for harness engineering.
- Make the best patterns portable across Python, TypeScript, Rust, and other stacks.
- Help people build their own tooling instead of copying surfaces blindly.

## Read by system

<div class="hg-grid hg-grid-2" markdown>

<div class="hg-panel" markdown>
### Blueprint chapters
- [Tool System](blueprint/chapters/tool-system.md)
- [Tool Execution Pipeline](blueprint/chapters/tool-execution-pipeline.md)
- [Query Engine](blueprint/chapters/query-engine.md)
- [Bootstrap](blueprint/chapters/bootstrap.md)
- [Permissions](blueprint/chapters/permissions.md)
- [Session and State](blueprint/chapters/session-and-state.md)
</div>

<div class="hg-panel" markdown>
### More systems
- [Multi-Agent Orchestration](blueprint/chapters/multi-agent-orchestration.md)
- [MCP Integration](blueprint/chapters/mcp-integration.md)
- [Slash Commands](blueprint/chapters/slash-commands.md)
- [Terminal UI](blueprint/chapters/terminal-ui.md)
- [IDE Bridge](blueprint/chapters/ide-bridge.md)
- [Memory and Cost](blueprint/chapters/memory-and-cost.md)
</div>

</div>

## Build better

These pages translate the blueprint into concrete design principles for new projects:

- fail closed by default
- make permissions explicit
- separate validation from policy
- preserve prompt-cache stability
- keep docs useful to builders, not spectators

## Reading tracks

<div class="hg-grid hg-grid-3" markdown>

<div class="hg-panel" markdown>
### If you are building from scratch
Start with harness fundamentals, then read Tool System, Query Engine, Permissions, and Session State in that order.
</div>

<div class="hg-panel" markdown>
### If you already have an agent
Go straight to Tool Execution Pipeline, Bootstrap, MCP, and IDE Bridge to tighten the runtime around what you already ship.
</div>

<div class="hg-panel" markdown>
### If you want docs that spread
Use the blueprint pages as source material, then publish implementation guides, diagrams, comparison pages, and contribution notes around them.
</div>

</div>

## Who this is for

- developers building AI coding agents
- toolsmiths designing safe execution harnesses
- open-source maintainers documenting agent internals
- researchers comparing runtime architectures across products

## Recommended reading path

1. Start with [What Is An Agent Harness?](start-here/what-is-an-agent-harness.md)
2. Read [How To Read The Blueprint](start-here/reading-the-blueprint.md)
3. Dive into [Blueprint Overview](blueprint/index.md)
4. Use [Build Better Principles](build-better/principles.md) when shaping your own implementation

## Why this can travel

Technical docs spread when they do three things well:

- name the moving parts clearly
- compress a hard system into reusable patterns
- help builders ship better work immediately

That is the standard for this site.

## Status

This site is structured to grow into a proper public docs hub. The blueprint is already here in both chapter pages and the complete long-form reference. The next layer is more synthesis, diagrams, comparative guides, and implementation walkthroughs.
