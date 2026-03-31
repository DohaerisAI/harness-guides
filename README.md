# Harness Guides

Build better AI agent harnesses.

Live site: https://dohaerisai.github.io/harness-guides/

Harness Guides is a public documentation project for engineers studying how serious agent runtimes are actually built. It focuses on the runtime layer around the model: tools, permissions, execution pipelines, query loops, state, sessions, multi-agent orchestration, MCP, bridge protocols, memory, and cost tracking.

The goal is simple: turn dense harness reverse-engineering into docs people can read, reuse, and build from.

## What is here

- A full long-form blueprint for building an AI agent CLI
- Chapter-based navigation across the major runtime systems
- Builder-first design principles for making harnesses safer and more durable
- A live docs site built with `MkDocs Material` and deployed on GitHub Pages

## Start reading

- [Live documentation site](https://dohaerisai.github.io/harness-guides/)
- [Blueprint overview](https://dohaerisai.github.io/harness-guides/blueprint/)
- [Full blueprint](https://dohaerisai.github.io/harness-guides/blueprint/full-blueprint/)
- [Build better principles](https://dohaerisai.github.io/harness-guides/build-better/principles/)

## Who this is for

- developers building AI coding agents
- toolsmiths designing execution harnesses
- researchers comparing agent runtime architectures
- open-source maintainers documenting agent internals

## Why it matters

Most agent products do not fail because the model answered one question badly. They fail because the runtime is vague:

- tools are exposed inconsistently
- permissions are confusing
- retries duplicate work
- state becomes hard to reason about
- context grows without discipline
- sub-agents and external tools expand faster than the harness can explain itself

Good harnesses feel smarter because the surrounding system is coherent.

## Repository structure

```text
.
├── docs/                    # MkDocs content
├── .github/workflows/       # GitHub Pages deployment workflow
├── mkdocs.yml               # Site configuration
├── requirements.txt         # Docs dependencies
└── README.md
```

## Local development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

## Build

```bash
mkdocs build
```

## Deploy

Push to `main` and let the GitHub Actions workflow in `.github/workflows/docs.yml` publish the site to GitHub Pages.

## Current direction

The site already includes the full blueprint and chapter navigation. The next layer is more synthesis:

- better diagrams
- implementation walkthroughs
- comparative harness patterns
- contribution-oriented docs for future collaborators
