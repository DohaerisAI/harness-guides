# Harness Guides

Build better AI agent harnesses.

<div class="hg-hero" markdown>

<p class="hg-kicker">Runtime architecture for serious agent builders</p>

<h2>Claude Code got exposed. Here’s the harness blueprint Python builders actually need.</h2>

<p class="hg-lede">After the March 31, 2026 reporting, a lot of people wanted to know how serious agent runtimes actually work. Harness Guides turns that moment into practical, Python-first documentation for building better harnesses, with OpenAI's Responses API as the concrete reference model layer.</p>

<div class="hg-signal-row" markdown>

<span class="hg-signal">12 architecture chapters</span>
<span class="hg-signal">python-first examples</span>
<span class="hg-signal">responses api mapping</span>

</div>

<div class="hero-actions" markdown>

[Read the Blueprint](blueprint/full-blueprint.md){ .md-button .md-button--primary }
[Start Building](blueprint/index.md){ .md-button }
[Python Path](start-here/python-path.md){ .md-button }

</div>

</div>

<div class="hg-grid hg-grid-3" markdown>

<div class="hg-panel" markdown>
### 12 core systems
From tools and permissions to MCP, bridge protocols, memory, and cost tracking.
</div>

<div class="hg-panel" markdown>
### Python-first docs
Every core chapter explains the system in Python terms before pointing back to provenance or portability notes.
</div>

<div class="hg-panel" markdown>
### Responses API grounded
OpenAI-specific examples map to the official Responses API instead of legacy chat-completions thinking.
</div>

</div>

## Why this exists now

On March 31, 2026, public reports circulated that Claude Code internals had been exposed again through npm packaging artifacts. That pushed a lot more developers to ask the right question:

not "how do I copy one product?"

but "how do serious agent harnesses actually work?"

Harness Guides exists to answer the second question. The goal is to study the architecture, extract the durable patterns, and turn them into public documentation that helps people build better systems intentionally.

## Python first

The current version of the site is optimized for people building harnesses in Python.

- chapter pages explain the architecture in Python terms first
- OpenAI examples use the Responses API as the concrete reference
- the original long-form blueprint remains available as provenance, not the primary teaching surface
- Rust can still be added later, but it is not required to make the guide useful now

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
- Make the best patterns practical for Python builders first.
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

## OpenAI reference layer

The site uses the official Responses API as the default OpenAI reference surface for:

- tool calls
- long-running work
- streaming responses
- conversation continuity
- built-in tools and remote MCP

Start here:

- [OpenAI Responses API](blueprint/openai-responses-api.md)
- [Python Path](start-here/python-path.md)

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
Start with the Python path, then read Tool System, Query Engine, Permissions, and Session State in that order.
</div>

<div class="hg-panel" markdown>
### If you already have an agent
Go straight to Tool Execution Pipeline, Bootstrap, MCP, and IDE Bridge to tighten the runtime around what you already ship in Python.
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
2. Read [Python Path](start-here/python-path.md)
3. Use [OpenAI Responses API](blueprint/openai-responses-api.md) for the model and tool layer
4. Dive into [Blueprint Overview](blueprint/index.md)
5. Use [Build Better Principles](build-better/principles.md) when shaping your own implementation

## Why this can travel

Technical docs spread when they do three things well:

- name the moving parts clearly
- compress a hard system into reusable patterns
- help builders ship better work immediately

That is the standard for this site.

## Status

This site is structured to grow into a proper public docs hub. The blueprint is already here in chapter pages and the complete long-form reference, but the reader path is now Python-first and implementation-oriented.
