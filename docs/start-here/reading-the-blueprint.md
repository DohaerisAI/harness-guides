# How To Read The Blueprint

The blueprint is long on purpose. It is closer to an engineering handbook than a blog post.

## Best way to use it

Read it in passes:

1. First pass: scan the chapter headings and understand the system boundaries.
2. Second pass: focus on the runtime slices you need right now, such as tools, query loop, or permissions.
3. Third pass: extract reusable patterns into your own implementation notes.

## What the blueprint is good for

- understanding how a real agent CLI is wired
- seeing where architectural complexity actually lives
- learning what belongs in the harness instead of the prompt
- identifying patterns worth porting into your own stack

## What it is not

- a promise that one source tree is the only correct design
- a substitute for your own product constraints
- a direct recipe for every language and deployment model

## Reading advice

Do not get trapped in surface imitation. Copying command names or UI details is low leverage. The important parts are:

- contracts
- execution order
- safety boundaries
- recovery behavior
- state ownership
- prompt and tool stability

Those are the pieces that make a harness durable.
