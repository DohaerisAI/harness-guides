# Build Better Principles

These are the principles that repeatedly show up in robust harness design.

## The short version

- default toward safety
- make runtime state legible
- keep permissions inspectable
- design for recovery before you need it
- document the architecture, not just the commands

## Fail closed

New tools should default toward caution. If concurrency safety is unknown, serialize. If mutability is unclear, treat the tool as mutating until proven otherwise.

## Separate validation from permission checks

Malformed input is a different problem from disallowed behavior. Keep those decisions separate so operators and contributors can reason about the system.

## Keep the loop explicit

Your query engine should make the control loop legible. Tool call, result injection, retry, recovery, and budget checks should not be hidden behind vague abstractions.

## Optimize for prompt stability

Tool lists, system prompt fragments, and shared context should be deterministic whenever possible. Stable prompt prefixes improve cacheability, cost, and latency.

## Preserve traceability

A good harness makes it easy to answer:

- what ran
- why it ran
- what permissions allowed it
- what state changed
- how the system recovered if something failed

## Document the runtime, not just the CLI

Most open-source projects document commands and install steps but skip runtime architecture. Builders need both.

## What to steal immediately

- split input validation from authorization logic
- stabilize tool ordering before prompt generation
- persist user messages before expensive calls
- keep sub-agent delegation bounded and inspectable
- connect budgeting to the actual loop, not just billing dashboards
