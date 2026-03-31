# The Agent Harness Blueprint

**How to Build an AI Agent CLI from Scratch — Patterns from Claude Code**

This document is a comprehensive engineering tutorial for developers who want to build their own AI agent harness — a system like Claude Code, Cursor, or Codex that wraps an LLM API with tool execution, permission control, streaming, session management, and a conversation loop. Every pattern here is extracted from Claude Code's real TypeScript source. If you can read TypeScript and have called an LLM API at least once, you have enough background to follow along and build your own.

## Master Architecture

```
User Input --> Bootstrap --> QueryEngine --> queryLoop()
                                                |
                           +--------+----------++---------+-----------+
                           |        |          |          |           |
                        API Call  Tools    Permissions  State     Sessions
                           |        |          |          |           |
                        Stream   Dispatch    Rules     AppState    Persist
                        Events   Pipeline   +Classify   Store     Transcript
                           |        |          |          |           |
                           +--------+----+-----+----------+-----------+
                                         |
                                   tool_result
                                   messages
                                    back to
                                  queryLoop()
```

## Conventions

- **Source:** references point to TypeScript files in Claude Code's source tree.
- All code snippets are real TypeScript, cleaned up for readability (imports, UI rendering, and analytics stripped).
- "Build It Yourself" sections at the end of each chapter give a minimal recipe you can implement in any language.

---

# Chapter 1: Tool System

> Tools are the agent's hands. Everything the agent does in the real world — reading files, running commands, searching code — goes through the tool interface.

## 1.1 The Tool Interface

Every tool in the system implements this interface. The methods below are the essential contract — rendering, analytics, and documentation noise have been stripped.

```typescript
interface Tool {
  // Identity
  name: string
  aliases?: string[]
  searchHint?: string

  // Schemas
  inputSchema: ZodSchema
  outputSchema?: ZodSchema

  // Core execution
  call(
    input: unknown,
    context: ToolUseContext,
    canUseTool: CanUseTool,
    parentMessage: AssistantMessage,
    onProgress?: (progress: ProgressEvent) => void
  ): Promise<ToolResult>

  // Permission & validation gates
  checkPermissions(
    input: unknown,
    context: ToolUseContext
  ): Promise<PermissionResult>

  validateInput(
    input: unknown,
    context: ToolUseContext
  ): Promise<ValidationResult>

  // Capability flags
  isEnabled(): boolean
  isReadOnly(input: unknown): boolean
  isConcurrencySafe(input: unknown): boolean
  isDestructive(input: unknown): boolean

  // Output control
  maxResultSizeChars: number

  // Result formatting
  mapToolResultToToolResultBlockParam(
    content: ToolResultContent,
    toolUseID: string
  ): ToolResultBlockParam
}
```

Key observations:

- **`call()` receives a `canUseTool` callback** — tools can spawn sub-tool-calls (the Agent tool does this), and the permission system flows through.
- **`onProgress` is optional** — long-running tools (Bash, Agent) emit intermediate progress; simple tools (Glob, Grep) return in one shot.
- **`checkPermissions` is separate from `validateInput`** — validation catches malformed input, permissions check policy. They run in sequence, never merged.

## 1.2 buildTool() Factory

Tools are defined as partial objects and completed by `buildTool()`, which spreads fail-closed defaults:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: "allow", updatedInput: input }),
  toAutoClassifierInput: () => "",
}

function buildTool(def: Partial<Tool> & { name: string }): Tool {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def }
}
```

**Why fail-closed defaults matter:**

| Default | Value | Effect |
|---------|-------|--------|
| `isConcurrencySafe` | `false` | New tools run serially until proven safe |
| `isReadOnly` | `false` | New tools are treated as mutating |
| `isDestructive` | `false` | Not flagged as dangerous (opt-in) |
| `checkPermissions` | `allow` | Open by default — tightened at registration |

The caller (tool author) spreads their overrides on top. A tool that only defines `name`, `inputSchema`, and `call()` gets safe defaults for everything else. This means forgetting to set `isConcurrencySafe` results in serial execution, not accidental parallel mutation — the system fails toward safety.

## 1.3 Tool Registration

### getAllBaseTools()

Built-in tools are registered in a single function. Feature-gated tools use conditional spreads:

```typescript
function getAllBaseTools(): Tool[] {
  return [
    AgentTool,
    BashTool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    WebFetchTool,
    WebSearchTool,
    SkillTool,
    ...(isWorktreeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(isTodoV2() ? [TaskCreateTool, TaskUpdateTool, TaskListTool] : []),
    ...(isToolSearchEnabled() ? [ToolSearchTool] : []),
  ]
}
```

This pattern keeps the tool list declarative. Feature flags gate entire tools — there is no runtime `if` inside each tool checking whether it should exist.

### assembleToolPool()

The final tool list merges built-in and MCP (Model Context Protocol) tools:

```typescript
function assembleToolPool(
  permissionContext: PermissionContext,
  mcpTools: Tool[]
): Tool[] {
  // Built-in tools, filtered by deny rules and isEnabled()
  const builtIn = getTools(permissionContext)

  // MCP tools, also filtered by deny rules
  const allowedMcp = filterToolsByDenyRules(mcpTools, permissionContext)

  // Sort each partition for prompt-cache stability, built-ins first
  return uniqBy(
    [...builtIn].sort(byName).concat(allowedMcp.sort(byName)),
    "name"
  )
}
```

**Why sort for cache stability?** The tool list is serialized into the system prompt. If tools appear in a different order between calls, the prompt changes, and the API's prompt cache misses. Sorting built-ins first (alphabetically) and MCP tools second (alphabetically) ensures the same tool set always produces the same prompt prefix. `uniqBy` on `name` means a built-in tool always wins over an MCP tool with the same name.

## 1.4 Reference Implementation: GlobTool

Here is GlobTool — one of the simplest real tools — cleaned up to show the pattern without noise:

```typescript
const GlobTool = buildTool({
  name: "Glob",

  inputSchema: z.object({
    pattern: z.string().describe("Glob pattern to match files against"),
    path: z
      .string()
      .optional()
      .describe("Directory to search in. Defaults to cwd."),
  }),

  outputSchema: z.object({
    filenames: z.array(z.string()),
    truncated: z.boolean(),
  }),

  isReadOnly: () => true,
  isConcurrencySafe: () => true,

  async checkPermissions(input, context) {
    return checkReadPermissionForTool(
      input.path ?? context.workingDirectory,
      context
    )
  },

  async validateInput(input, context) {
    const targetPath = input.path ?? context.workingDirectory
    if (!existsSync(targetPath)) {
      return { result: false, message: `Path does not exist: ${targetPath}` }
    }
    const stat = statSync(targetPath)
    if (!stat.isDirectory()) {
      return { result: false, message: `Path is not a directory: ${targetPath}` }
    }
    return { result: true }
  },

  async call(input, context) {
    const cwd = input.path ?? context.workingDirectory
    const matches = await glob(input.pattern, {
      cwd,
      nodir: true,
      absolute: true,
      maxResults: MAX_GLOB_RESULTS,
    })

    // Sort by modification time, most recent first
    const sorted = matches
      .map((f) => ({ file: f, mtime: statSync(f).mtimeMs }))
      .sort((a, b) => b.mtime - a.mtime)
      .map((entry) => entry.file)

    const truncated = sorted.length >= MAX_GLOB_RESULTS
    return {
      data: { filenames: sorted, truncated },
    }
  },

  mapToolResultToToolResultBlockParam(content, toolUseID) {
    const { filenames, truncated } = content.data
    const text = filenames.join("\n") + (truncated ? "\n(results truncated)" : "")
    return {
      type: "tool_result",
      tool_use_id: toolUseID,
      content: text || "No files matched the pattern.",
    }
  },
})
```

**What to notice:**

- `isReadOnly: true` and `isConcurrencySafe: true` — multiple Glob calls can run in parallel, and they never mutate the filesystem.
- `checkPermissions` delegates to a shared read-permission helper — it does not reinvent permission logic.
- `validateInput` checks preconditions (path exists, is a directory) before `call()` runs.
- `call()` returns `{ data: ... }` — the data is typed by `outputSchema`.
- `mapToolResultToToolResultBlockParam` controls what the model sees — it joins filenames with newlines, a format the model parses easily.

## 1.5 Build It Yourself

**5-step recipe for a minimal tool system:**

1. **Define a Tool protocol/interface** with at minimum: `name`, `inputSchema`, `call()`, `checkPermissions()`. Add `isReadOnly()` and `isConcurrencySafe()` from the start — you will need them for the executor.

2. **Create a `buildTool()` factory** that applies fail-closed defaults. Every boolean capability defaults to the safe value (not concurrent, not read-only). Every permission check defaults to allow (tightened at the pool level).

3. **Register tools in a flat array.** Use feature flags with conditional spreads for optional tools. Do not use a registry class or dependency injection — a function returning an array is simpler and easier to debug.

4. **Write `assembleToolPool()`** that merges built-in tools with external (MCP) tools. Sort deterministically for prompt-cache stability. Deduplicate by name, built-ins winning.

5. **Start with GlobTool-level simplicity.** Your first tool should have a Zod schema (or equivalent), a permission check that delegates to a shared helper, and a single async function body. Get one tool working end-to-end before building complex ones.

```
ToolDef (partial) --> buildTool() --> Tool (complete)
                                         |
                       getAllBaseTools() --> [Tool, Tool, ...]
                                         |
                       getTools(permCtx) --> filter deny rules + isEnabled
                                         |
                       assembleToolPool() --> dedup, sort, merge MCP
```

---

# Chapter 2: Tool Execution Pipeline

> Between the model saying "use this tool" and the tool actually running, six phases of validation, permission, and hook processing ensure nothing executes without authorization.

## 2.1 The 6-Phase Pipeline

Every tool call passes through these phases in order. Failure at any phase short-circuits — the remaining phases do not run.

**Phase 1: Schema Validation**

```typescript
const parsed = tool.inputSchema.safeParse(input)
if (!parsed.success) {
  return createErrorResult(toolUseID, formatZodError(parsed.error))
}
```

The model sometimes sends malformed JSON or missing required fields. Schema validation catches this before any side effects occur. The error message goes back to the model so it can retry with correct input.

**Phase 2: Semantic Validation**

```typescript
const valid = await tool.validateInput(parsed.data, context)
if (!valid.result) {
  return createErrorResult(toolUseID, valid.message)
}
```

Schema validation checks structure; semantic validation checks meaning. "Does this file path exist?" "Is this a directory, not a file?" "Is the timeout within bounds?" These are business rules the schema cannot express.

**Phase 3: Pre-Tool Hooks**

```typescript
for await (const result of runPreToolUseHooks(context, tool, input)) {
  if (result.hookPermissionResult) {
    // Hook can approve or deny the tool call
    hookDecision = result.hookPermissionResult
  }
  if (result.updatedInput) {
    // Hook can transform input (e.g., rewrite paths, add defaults)
    input = result.updatedInput
  }
}
```

Hooks are user-defined code that runs before and after tools. A pre-tool hook can approve a tool call (skipping the interactive permission prompt), deny it outright, or transform the input. Hooks are configured in project settings and can implement auto-accept rules, audit logging, or input sanitization.

**Phase 4: Permission Decision**

```typescript
const decision = await resolvePermission(hookResult, tool, input, canUseTool)
if (decision.behavior !== "allow") {
  return createDenialResult(toolUseID, decision.message)
}
```

This phase resolves the final permission. Sources of permission (in priority order): hook approval/denial, tool's own `checkPermissions()`, interactive user prompt via `canUseTool()`. If no hook has decided, and `checkPermissions()` returns `ask`, the system calls `canUseTool()` to prompt the user.

**Phase 5: Tool Execution**

```typescript
const result = await tool.call(input, context, canUseTool, parentMessage, onProgress)
```

The tool runs. This is the only phase with side effects. `onProgress` streams intermediate output for long-running tools. `canUseTool` is passed through so tools that spawn sub-tools (like the Agent tool) can request permissions for those sub-calls.

**Phase 6: Post-Tool Hooks**

```typescript
for await (const hookResult of runPostToolUseHooks(context, tool, result)) {
  if (hookResult.updatedMCPToolOutput) {
    result = hookResult.updatedMCPToolOutput
  }
}
```

Post-tool hooks can transform the output before it is sent back to the model. Use cases: auto-formatting code output, redacting sensitive data, adding metadata.

## 2.2 runToolUse() -- The Async Generator

The pipeline is wrapped in an async generator so progress events and the final result flow through the same channel:

```typescript
async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseTool,
  context: ToolUseContext
): AsyncGenerator<ToolUpdate> {
  // Find tool by name (or alias)
  const tool = findToolByName(context.tools, toolUse.name)
  if (!tool) {
    yield {
      message: createErrorResult(toolUse.id, `No such tool: ${toolUse.name}`),
    }
    return
  }

  // Run the 6-phase pipeline, yielding progress along the way
  for await (const update of checkPermissionsAndCallTool(
    tool,
    toolUse.id,
    toolUse.input,
    assistantMessage,
    canUseTool,
    context
  )) {
    yield update
  }
}
```

**Why an async generator?** The tool execution pipeline produces multiple events over time: permission prompts, progress updates, partial output, and the final result. An async generator naturally models this sequence without callbacks or event emitters. The caller (`StreamingToolExecutor`) consumes updates as they arrive and forwards them to the UI.

## 2.3 StreamingToolExecutor

The model can request multiple tool calls in a single response. The `StreamingToolExecutor` manages concurrent execution with a safety invariant: **tools marked `isConcurrencySafe` run in parallel; all others run exclusively.**

```typescript
class StreamingToolExecutor {
  private tools: TrackedTool[] = []

  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage) {
    const tool = findToolByName(this.allTools, block.name)
    const isConcurrencySafe = tool?.isConcurrencySafe(block.input) ?? false
    this.tools.push({
      id: block.id,
      name: block.name,
      input: block.input,
      status: "queued",
      isConcurrencySafe,
      assistantMessage,
    })
    this.processQueue()
  }

  private canExecuteTool(isConcurrencySafe: boolean): boolean {
    const executing = this.tools.filter((t) => t.status === "executing")
    if (executing.length === 0) return true
    // Parallel only if ALL executing tools AND the new tool are concurrency-safe
    return isConcurrencySafe && executing.every((t) => t.isConcurrencySafe)
  }

  private processQueue() {
    for (const tool of this.tools) {
      if (tool.status !== "queued") continue
      if (this.canExecuteTool(tool.isConcurrencySafe)) {
        tool.status = "executing"
        this.executeTool(tool)
      } else if (!tool.isConcurrencySafe) {
        break // Must wait — this tool needs exclusive access
      }
      // If concurrency-safe but blocked by a non-safe tool, skip and check next
    }
  }

  private async executeTool(tracked: TrackedTool) {
    for await (const update of runToolUse(tracked, ...)) {
      tracked.updates.push(update)
    }
    tracked.status = "completed"
    this.processQueue() // Unblock waiting tools
  }

  getCompletedResults(): ToolUpdate[] {
    return this.tools
      .filter((t) => t.status === "completed")
      .flatMap((t) => t.updates)
  }
}
```

**Concurrency rules:**

| Currently executing | New tool | Action |
|---|---|---|
| Nothing | Any | Execute immediately |
| Concurrency-safe tools only | Concurrency-safe | Execute in parallel |
| Concurrency-safe tools only | Non-safe | Wait for all to finish |
| Non-safe tool | Any | Wait for it to finish |

This means three concurrent `Glob` calls (all `isConcurrencySafe: true`) run in parallel, but a `BashTool` call (`isConcurrencySafe: false`) waits for exclusive access. The queue processes in order, so a non-safe tool blocks everything behind it.

## 2.4 Progress Callbacks

Long-running tools use `onProgress` to stream intermediate output to the UI. Here is the pattern from BashTool:

```typescript
async call(input, context, canUseTool, parentMessage, onProgress) {
  const generator = runShellCommand({
    command: input.command,
    timeout: input.timeout,
    cwd: context.workingDirectory,
    abortSignal: context.abortSignal,
  })

  let counter = 0
  let result

  do {
    result = await generator.next()
    if (!result.done && onProgress) {
      onProgress({
        toolUseID: `bash-progress-${counter++}`,
        data: {
          type: "bash_progress",
          output: result.value.output,
          isStderr: result.value.isStderr,
        },
      })
    }
  } while (!result.done)

  return {
    data: {
      exitCode: result.value.exitCode,
      stdout: result.value.stdout,
      stderr: result.value.stderr,
    },
  }
}
```

The shell command itself is an async generator that yields output chunks as they arrive from the subprocess. BashTool consumes these chunks and re-emits them as progress events. The UI renders them as live terminal output. When the command finishes, BashTool returns the final result with exit code.

## 2.5 Build It Yourself

**4-step recipe for a minimal execution pipeline:**

1. **Implement the 6-phase pipeline as a function.** Each phase returns early on failure. Start with just schema validation + call; add permission checks and hooks incrementally.

2. **Wrap the pipeline in an async generator** (or your language's equivalent — channels in Go, streams in Rust). This gives you a single return type for progress events, permission prompts, and the final result.

3. **Build `StreamingToolExecutor`** with a queue and concurrency check. The invariant is simple: count executing tools, check if all are concurrency-safe. Start with serial-only execution and add parallelism once your tests pass.

4. **Add `onProgress` to your tool interface** but make it optional. Only long-running tools (shell commands, agent sub-calls) need it. Short tools (file read, glob) return immediately.

```
tool_use block from API
        |
  runToolUse() <-- async generator
        |
  6-phase pipeline
    1. Schema validate
    2. Semantic validate
    3. Pre-tool hooks
    4. Permission decision
    5. Tool execution  <-- side effects here
    6. Post-tool hooks
        |
  tool_result message --> back to query loop
```

---

# Chapter 3: Query Engine & Conversation Loop

> The query engine is the brain's control loop: send messages to the API, collect tool calls, execute them, feed results back, and repeat until the model stops or a budget is exhausted.

## 3.1 QueryEngine Class

The `QueryEngine` owns the conversation state and provides the entry point for all interactions:

```typescript
class QueryEngine {
  private mutableMessages: Message[]
  private totalUsage: Usage
  private permissionDenials: PermissionDenial[]
  private readFileState: FileStateCache

  constructor(config: QueryEngineConfig) {
    this.mutableMessages = config.initialMessages ?? []
    this.totalUsage = {
      input_tokens: 0,
      output_tokens: 0,
      cache_creation_input_tokens: 0,
      cache_read_input_tokens: 0,
    }
    this.permissionDenials = []
    this.readFileState = config.readFileCache
  }
}
```

**Key state:**

- **`mutableMessages`** — the full conversation history. This is the one piece of mutable state in the system (necessary because the API expects the full history on each call).
- **`totalUsage`** — accumulated token counts across all turns, used for budget enforcement.
- **`permissionDenials`** — tracks which permissions the user denied, so the system does not re-ask within the same session.
- **`readFileState`** — caches file contents so the system can detect when a file has changed between reads (stale read detection).

## 3.2 submitMessage() Entry Point

`submitMessage()` is the public API. It takes a user prompt and yields messages as they arrive:

```typescript
async *submitMessage(
  prompt: string,
  options?: SubmitOptions
): AsyncGenerator<NormalizedMessage> {
  // 1. Build system prompt from context
  const { systemPrompt, userContext, systemContext } =
    await fetchSystemPromptParts(this.config, this.readFileState)

  // 2. Process user input (check for slash commands like /help, /clear)
  const { messages, shouldQuery } = await processUserInput(
    prompt,
    this.mutableMessages,
    this.config
  )
  this.mutableMessages.push(...messages)

  // 3. Early return for slash commands that don't need the API
  if (!shouldQuery) {
    yield { type: "command_result", content: messages }
    return
  }

  // 4. Persist transcript BEFORE API call (crash recovery)
  await recordTranscript(this.mutableMessages)

  // 5. Enter query loop
  const tools = assembleToolPool(this.config.permissionContext, this.mcpTools)
  for await (const message of queryLoop({
    messages: this.mutableMessages,
    systemPrompt,
    tools,
    canUseTool: this.canUseTool.bind(this),
    maxTurns: this.config.maxTurns,
    maxBudgetUsd: this.config.maxBudgetUsd,
    abortSignal: this.config.abortSignal,
  })) {
    // 6. Accumulate usage, track messages
    this.totalUsage = accumulateUsage(this.totalUsage, message.usage)
    this.mutableMessages.push(message)
    yield normalizeMessage(message)
  }

  // 7. Yield final result with totals
  yield {
    type: "result",
    usage: this.totalUsage,
    stop_reason: this.lastStopReason,
    cost_usd: calculateCost(this.totalUsage),
  }
}
```

**Why persist before the API call?** If the process is killed during a long API stream, the transcript file contains the user's message. On restart with `--resume`, the system loads the transcript and replays from the last user message. If persistence happened after the API call, a crash would lose the user's prompt entirely.

## 3.3 queryLoop() -- The Heart

This is the central loop. It calls the API, executes tools, handles errors, and decides whether to continue or stop.

```typescript
type QueryLoopState = {
  messages: Message[]
  toolUseContext: ToolUseContext
  turnCount: number
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
}

async function* queryLoop(
  params: QueryLoopParams
): AsyncGenerator<Message> {
  let state: QueryLoopState = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    turnCount: 1,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
  }

  while (true) {
    const { messages, turnCount } = state

    // ── Budget gates ──
    if (turnCount >= params.maxTurns) {
      yield createErrorMessage("error_max_turns")
      return
    }
    if (params.totalCostUsd >= params.maxBudgetUsd) {
      yield createErrorMessage("error_max_budget")
      return
    }
    if (params.abortSignal?.aborted) {
      return
    }

    // ── 1. STREAMING API CALL ──
    let needsFollowUp = false
    let withheld = false
    let stopReason: string | null = null
    const toolResults: ToolResultMessage[] = []
    const executor = new StreamingToolExecutor(state.toolUseContext.tools)

    for await (const event of callModel({
      messages,
      systemPrompt: params.systemPrompt,
      tools: params.tools,
      abortSignal: params.abortSignal,
    })) {
      if (event.type === "assistant") {
        // ── 2. COLLECT TOOL USE BLOCKS ──
        const toolBlocks = event.content.filter(
          (c) => c.type === "tool_use"
        )
        if (toolBlocks.length > 0) {
          needsFollowUp = true
          for (const block of toolBlocks) {
            executor.addTool(block, event)
          }
        }
        stopReason = event.stop_reason
      }

      // ── 3. YIELD COMPLETED TOOL RESULTS ──
      for (const result of executor.getCompletedResults()) {
        yield result.message
        toolResults.push(result.message)
      }

      // ── 4. WITHHOLD RECOVERABLE ERRORS ──
      if (isPromptTooLong(event)) {
        withheld = true
      }
      if (isMaxOutputTokens(event)) {
        withheld = true
      }
      if (!withheld) {
        yield event
      }
    }

    // Drain any remaining completed tools after stream ends
    for (const result of executor.getCompletedResults()) {
      yield result.message
      toolResults.push(result.message)
    }

    // ── 5. RECOVERY: prompt too long ──
    if (withheld && isPromptTooLong && !state.hasAttemptedReactiveCompact) {
      const compacted = await compactMessages(messages)
      state = {
        ...state,
        messages: compacted,
        hasAttemptedReactiveCompact: true,
      }
      continue // Retry with compressed history
    }

    // ── 5b. RECOVERY: max output tokens ──
    if (withheld && isMaxOutputTokens && state.maxOutputTokensRecoveryCount < 3) {
      state = {
        ...state,
        maxOutputTokensRecoveryCount: state.maxOutputTokensRecoveryCount + 1,
      }
      continue // Retry — model will continue where it left off
    }

    // ── 6. FEED TOOL RESULTS BACK ──
    if (needsFollowUp && turnCount < params.maxTurns) {
      const updatedMessages = [
        ...messages,
        createUserMessage({ content: toolResults }),
      ]
      state = {
        ...state,
        messages: updatedMessages,
        turnCount: turnCount + 1,
      }
      continue // Next API call with tool results
    }

    // ── 7. TERMINAL — no more tool calls, model is done ──
    return
  }
}
```

**The immutable state pattern:** Notice that `state` is reassigned, never mutated. Each iteration creates a new state object with the spread operator. This makes the loop's behavior predictable — you can inspect any state snapshot without worrying about hidden mutations from earlier in the iteration.

**The `continue`/`return` discipline:** Every branch at the bottom of the loop either `continue`s (more work to do) or `return`s (done). There is no fallthrough. This makes the control flow explicit and prevents accidental infinite loops.

## 3.4 Recovery Mechanisms

The query loop handles four categories of recoverable failure:

| Scenario | Detection | Recovery | Limit |
|---|---|---|---|
| Prompt too long | API returns 400 with `prompt_too_long` | Reactive compact: summarize older messages, keep recent context | 1 attempt |
| Max output tokens | `stop_reason === "max_tokens"` | Truncate partial output, retry — model continues from truncation point | 3 attempts |
| Context collapse | Token count crosses threshold | Drain oldest messages from history proactively | Feature-gated |
| Model fallback | Streaming error (network, server) | Insert tombstone message explaining gap, optionally switch model | 1 fallback |

**Reactive compaction** deserves detail: when the prompt exceeds the model's context window, the system summarizes the conversation history into a condensed form. Recent messages (last N turns) are kept verbatim. Older messages are replaced with a system-generated summary. This preserves the model's understanding of recent context while freeing token budget.

**Max output token recovery** works because the API returns partial output. The system keeps the partial output, appends it to the history, and calls the API again. The model sees its own partial response and continues from where it stopped. Three retries are allowed before the system gives up.

## 3.5 Budget Gates

Budget checks run at the top of each loop iteration, before the API call:

```typescript
// Turn limit — prevents runaway loops
if (turnCount >= maxTurns) {
  yield createErrorMessage("error_max_turns")
  return
}

// Dollar cost limit — prevents surprise bills
if (totalCostUSD >= maxBudgetUsd) {
  yield createErrorMessage("error_max_budget")
  return
}

// Token budget — finer-grained than dollar cost
if (tokenBudget.exceeded) {
  yield createErrorMessage("error_max_budget")
  return
}

// User cancellation — Ctrl+C or programmatic abort
if (abortSignal.aborted) {
  return
}

// Structured output retries — prevents infinite retry loops
if (structuredOutputRetries >= 5) {
  yield createErrorMessage("error_max_structured_output_retries")
  return
}
```

These gates are checked before every API call, so the system cannot spend more than one turn's worth of tokens over budget. The dollar cost is calculated from token counts using the model's pricing table.

## 3.6 Transcript Persistence Strategy

Transcript writes are strategically timed for crash resilience without sacrificing performance:

```
BEFORE query: user messages      --> BLOCKING write (~4ms on SSD)
DURING query: assistant messages --> fire-and-forget (non-blocking)
AFTER query:  final flush        --> BLOCKING write for safety
```

**The reasoning:**

- **User messages are irreplaceable** — they represent the human's intent. A blocking write ensures they are on disk before the API call starts. If the process crashes during the API call, `--resume` has the user's message and can replay.
- **Assistant messages are reproducible** — the API can regenerate them. Fire-and-forget writes are fast and usually succeed, but losing one is recoverable by re-calling the API.
- **Final flush is blocking** — at the end of a turn, the full conversation (including tool results) is persisted. This ensures `--resume` starts from a complete state.

The transcript file is append-only JSON Lines. Each line is a complete message object. This format survives partial writes — a crash mid-write loses at most one message, and the file parser skips malformed trailing lines.

## 3.7 Build It Yourself

**5-step recipe for a minimal query engine:**

1. **Start with a `while (true)` loop.** Call the API, check for tool_use blocks in the response, execute them, append results, and call the API again. Break when the response has no tool_use blocks.

2. **Add budget gates** at the top of the loop. At minimum: turn count limit and abort signal. Add dollar cost limits once you have token counting.

3. **Implement `submitMessage()`** as the public entry point. It builds the system prompt, appends the user message, enters the query loop, and returns the accumulated result.

4. **Add transcript persistence.** Write user messages to disk before the API call. Write everything else after. Use append-only JSON Lines — it is the simplest crash-safe format.

5. **Add recovery.** Start with max-output-token retry (easiest — just call the API again). Add reactive compaction later when you hit context window limits.

```
                       submitMessage(prompt)
                              |
                     build system prompt
                     append user message
                     persist transcript    <-- blocking write
                              |
                    +-- queryLoop() --+
                    |                 |
                    v                 |
               call API              |
                    |                 |
              tool_use?  ---no----> return result
                    |                 ^
                   yes                |
                    |                 |
              execute tools           |
              (6-phase pipeline)      |
                    |                 |
              append tool_results     |
              to messages             |
                    |                 |
              budget check            |
                    |                 |
                 continue  -----------+
```

---

# Chapter 4: Bootstrap & Startup

> The bootstrap pipeline gets the agent from a cold process to a warm query loop in under 500ms, using parallel I/O and deferred initialization to hide latency.

## 4.1 The 7-Stage Pipeline

```typescript
// ── Stage 1: Top-level prefetch ──
// These start BEFORE any imports finish loading.
// The goal is to overlap I/O with module initialization.
startMdmRawRead()       // MDM subprocess — reads device config in parallel
startKeychainPrefetch() // Keychain read (~65ms) — overlaps with import time

// ── Stage 2: Warning handler + environment guards ──
process.on("warning", suppressExperimentalWarnings)
if (getNodeMajorVersion() < 18) {
  exitWithError("Node.js 18+ required")
}
validatePlatform() // Linux, macOS, WSL — not raw Windows

// ── Stage 3: CLI parser + trust gate ──
const program = createCommand("claude")
  .option("--print", "Non-interactive mode")
  .option("--resume", "Resume last session")
  .option("--model", "Model override")
  .hook("preAction", authenticateUser) // Auth check before any action

// ── Stage 4: setup() + parallel loads ──
await Promise.all([
  getCommands(),        // Lazy-load all slash command definitions
  loadAgentsDir(),      // Discover agent YAML files from ~/.claude/agents/
  initSessionMemory(),  // Register memory hooks for session persistence
  connectMcpServers(),  // Start MCP server connections (can be slow)
])

// ── Stage 5: Deferred init after trust ──
if (userIsTrusted) {
  initPlugins()      // Load plugins from settings
  initSkills()       // Load skills from 5 directories:
                     //   project, user, agents, built-in, MCP
  initMcpPrefetch()  // Pre-fetch MCP tool schemas in background
  initHooks()        // Register PreToolUse, PostToolUse, Stop hooks
}

// ── Stage 6: Mode routing ──
switch (mode) {
  case "local":
    startLocalRepl(queryEngine)       // Standard interactive REPL
    break
  case "remote":
    startRemoteSession(queryEngine)   // Remote session control
    break
  case "ssh":
    startSshProxy(queryEngine)        // SSH tunneled session
    break
  case "teleport":
    resumeTeleportedSession(config)   // Resume from another machine
    break
  case "bridge":
    startBridgeMode(queryEngine)      // IDE extension communication
    break
}

// ── Stage 7: Query engine submit loop ──
if (singlePrompt) {
  // --print mode: one prompt, one response, exit
  for await (const msg of queryEngine.submitMessage(prompt)) {
    render(msg)
  }
  process.exit(0)
} else {
  // REPL mode: read prompt, submit, render, repeat
  while (true) {
    const input = await readUserInput()
    for await (const msg of queryEngine.submitMessage(input)) {
      render(msg)
    }
  }
}
```

**Latency hiding strategy:** Stages 1 and 4 are the key performance wins. By starting keychain reads and MDM subprocess reads before imports finish, the I/O completes during time that would otherwise be spent loading modules. Stage 4 uses `Promise.all` to load commands, agents, memory, and MCP connections in parallel — any one of these can take 50-200ms, but they overlap.

**Trust gating (Stage 5):** Plugins, skills, and hooks only load after authentication succeeds. This prevents untrusted code from executing during the bootstrap — a plugin cannot run before the user's identity is verified.

## 4.2 System Prompt Assembly

The system prompt is built from two memoized context functions. Memoization ensures expensive operations (git status, file system reads) run at most once per session:

```typescript
const getSystemContext = memoize(async (): Promise<SystemContext> => {
  const parts: SystemContext = {}

  // Git context — expensive subprocess call, cached
  const gitStatus = await getGitStatus()
  if (gitStatus) {
    parts.gitStatus = gitStatus
  }

  // Working directory, platform info
  parts.cwd = process.cwd()
  parts.platform = process.platform
  parts.shell = process.env.SHELL ?? "bash"

  return parts
})

const getUserContext = memoize(async (): Promise<UserContext> => {
  const parts: UserContext = {}

  // CLAUDE.md files — project instructions from multiple directories
  const memoryFiles = await getMemoryFiles()
  const claudeMds = getClaudeMds(memoryFiles)
  if (claudeMds.length > 0) {
    parts.claudeMd = claudeMds
  }

  // Current date — models do not know today's date
  parts.currentDate = `Today's date is ${getLocalISODate()}.`

  return parts
})
```

**Why two separate functions?** System context (git status, platform) changes rarely and is project-scoped. User context (CLAUDE.md files, date) is user-scoped and includes personal instructions. Separating them allows different cache invalidation strategies — system context can be refreshed on directory change, while user context stays stable within a session.

**CLAUDE.md resolution order:** The system searches for instruction files in multiple directories: the project root, parent directories up to the git root, the user's home directory, and the `~/.claude/` config directory. Files found in more specific locations (project root) take precedence over general ones (home directory). This lets users set global defaults that projects can override.

## 4.3 Build It Yourself

**3-step recipe for a minimal bootstrap:**

1. **Parse CLI arguments, then authenticate.** Use any CLI parser (Commander, clap, argparse). Check authentication before loading any user-defined code (plugins, hooks, skills). The auth check should be a "pre-action" hook on the CLI parser so it runs before any command handler.

2. **Load configuration in parallel.** Identify your independent I/O operations (reading config files, connecting to external servers, discovering plugins) and run them concurrently. Even in a simple system, `Promise.all([loadConfig(), loadTools(), loadInstructions()])` can save 100-300ms on startup.

3. **Build the system prompt from memoized functions.** Separate context into categories (system, user, project) with independent cache lifetimes. Assemble the final prompt by concatenating sections. Include the current date — models do not know what day it is.

```
Process start
     |
     +-- prefetch I/O (keychain, config)    } parallel with
     +-- import modules                      } module loading
     |
  CLI parse + auth gate
     |
  Promise.all([
     load commands,
     load agents,
     init memory,
     connect MCP
  ])
     |
  mode routing --> REPL or single-prompt
     |
  queryEngine.submitMessage() loop
```

---

# Chapter 5: Permission System

## 5.1 ToolPermissionContext

Show the type:
```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: 'default' | 'auto' | 'plan' | 'bypass'
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
}>
```

Rule sources in priority order:
```
policySettings > projectSettings > flagSettings > localSettings >
userSettings > cliArg > command > session
```

## 5.2 Permission Decision Chain

ASCII flow diagram:
```
Tool use request
    ↓
PreToolUse hooks → hook says allow/deny? → done
    ↓ (no decision)
Rule-based check (alwaysDeny > alwaysAllow > alwaysAsk)
    ↓ (no match)
Mode check:
  auto → Transcript classifier (side-query to small model)
  default → Interactive permission dialog
  plan → Deny all mutations
  bypass → Allow all
    ↓
PermissionDecision { behavior: allow|deny|ask, updatedInput?, reason? }
```

## 5.3 Auto-Mode Classifier

Show the concept:
```typescript
type AutoModeRules = {
  allow: string[]     // "git status", "npm test"
  soft_deny: string[] // "rm -rf", "DROP TABLE"
  environment: string[] // "Node.js project", "uses PostgreSQL"
}

// Side-query to Haiku with the tool input + rules
// Returns: { matches: boolean, confidence: 'high'|'low', matchedDescription? }
```

## 5.4 Bash Security

- AST parsing via tree-sitter (not regex)
- Dangerous pattern detection: Bash(*), python:*, node* wildcards
- UNC path blocking (prevents NTLM credential leaks)
- Device file blacklist (/dev/zero, /dev/stdin, /dev/tty)

## 5.5 Minimal Python Model

```python
@dataclass(frozen=True)
class ToolPermissionContext:
    deny_names: frozenset[str] = field(default_factory=frozenset)
    deny_prefixes: tuple[str, ...] = ()

    def blocks(self, tool_name: str) -> bool:
        lowered = tool_name.lower()
        return lowered in self.deny_names or any(
            lowered.startswith(p) for p in self.deny_prefixes
        )
```

## 5.6 Build It Yourself

4-step recipe:
1. Define a PermissionContext with deny-lists (start simple like the Python model)
2. Add a rule-based checker with source priority
3. Implement an interactive dialog fallback for "ask" decisions
4. Optionally add a classifier for auto-mode (side-query to fast model)

---

# Chapter 6: Session & State Management

## 6.1 AppState — Immutable Store

Show the core shape:
```typescript
type AppState = DeepImmutable<{
  settings: SettingsJson
  mainLoopModel: ModelSetting
  toolPermissionContext: ToolPermissionContext
  verbose: boolean
  agent: string | undefined

  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    resources: Record<string, ServerResource[]>
  }

  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
  }

  tasks: Record<string, TaskState>
}>

// Zustand-like store
type Store<T> = {
  getState(): T
  setState(fn: (prev: T) => T): void
  subscribe(listener: (state: T) => void): () => void
}
```

## 6.2 React Context Integration

```typescript
// Provider wraps entire app
<AppStateProvider store={store}>
  <REPL />
</AppStateProvider>

// Components access state via hooks
function StatusLine() {
  const model = useAppState(s => s.mainLoopModel)
  const cost = useAppState(s => s.totalCost)
  return <Text>{model} | ${cost.toFixed(4)}</Text>
}
```

## 6.3 Context Compression

Show the constants and strategy:
```typescript
const POST_COMPACT_TOKEN_BUDGET    = 50_000  // total budget for restored context
const POST_COMPACT_MAX_FILES       = 5       // max files to restore
const POST_COMPACT_MAX_PER_FILE    = 5_000   // tokens per restored file
const POST_COMPACT_MAX_PER_SKILL   = 5_000   // tokens per restored skill
const POST_COMPACT_SKILLS_BUDGET   = 25_000  // total budget for skills

// Compact flow:
// 1. Strip images from messages (prevent prompt-too-long in compact call)
// 2. Send to compaction API → receive summary
// 3. Replace pre-compact messages with summary + boundary marker
// 4. Restore top-N files and skills within token budget
// 5. Release pre-compact memory for GC (splice mutableMessages)
```

## 6.4 Transcript Persistence

```
Write-before-query pattern:
  ┌─────────────────────────────────────────────┐
  │ User message → recordTranscript() [BLOCK]   │  ← crash recovery
  │ API call starts...                          │
  │ Assistant streams → void recordTranscript() │  ← fire-and-forget
  │ Tool results...                             │
  │ Turn complete → flushSessionStorage()       │  ← if EAGER_FLUSH
  └─────────────────────────────────────────────┘
```

Session save/load:
```python
# Python minimal model
@dataclass(frozen=True)
class StoredSession:
    session_id: str
    messages: tuple[str, ...]
    input_tokens: int
    output_tokens: int

def save_session(session, directory):
    path = directory / f'{session.session_id}.json'
    path.write_text(json.dumps(asdict(session)))
    return path
```

## 6.5 Build It Yourself

4-step recipe.

---

# Chapter 7: Multi-Agent Orchestration

## 7.1 Agent Spawning

Show the AgentTool input schema:
```typescript
const inputSchema = z.object({
  description: z.string(),      // Short task description
  prompt: z.string(),           // The task for the agent
  subagent_type: z.string().optional(),  // Specialized agent type
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  isolation: z.enum(['worktree']).optional(),
})
```

Two execution modes:
```
Synchronous:  result = await runAgent(agentDef, prompt, subContext)
              return { status: 'completed', result: result.output }

Asynchronous: const agentId = createAgentId()
              registerAsyncAgent(agentId, () => runAgent(...))
              return { status: 'async_launched', agentId }
```

## 7.2 Coordinator Mode

ASCII diagram:
```
┌─────────────────────────────────────────┐
│ Coordinator (restricted tools)           │
│   Tools: Agent, SendMessage, TaskStop    │
│   Role: orchestrate, synthesize, report  │
├─────────────────────────────────────────┤
│   ↓ spawn        ↓ spawn       ↓ spawn  │
│ Worker 1       Worker 2      Worker 3    │
│ (full tools)  (full tools)  (full tools) │
│ Bash,Read,    Bash,Read,    Bash,Read,   │
│ Edit,Grep,    Edit,Grep,    Edit,Grep,   │
│ Glob,Web...   Glob,Web...   Glob,Web...  │
└─────────────────────────────────────────┘
```

Show the coordinator system prompt (condensed):
```
You are a coordinator. Your job is to:
- Direct workers to research, implement, and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible

Workers arrive as <task-notification> XML:
  <task-id>{agentId}</task-id>
  <status>completed|failed</status>
  <result>{agent's response}</result>

Parallelism is your superpower. Launch independent workers concurrently.
```

## 7.3 Prompt Cache Sharing

Show CacheSafeParams:
```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt       // Must match parent exactly
  userContext: Record<string, string>
  systemContext: Record<string, string>
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]   // Parent's message history
}

// Saved after each turn, read by forked agents
let lastCacheSafeParams: CacheSafeParams | null = null
```

Why: Sub-agents that share CacheSafeParams get cache hits on the parent's prompt prefix → saves tokens/money.

## 7.4 SendMessage Routing

```
to="worker-name"     → direct message to named agent
to="*"               → broadcast to all agents
to="uds:<socket>"    → Unix domain socket peer
to="bridge:<session>" → IDE bridge session
```

## 7.5 Build It Yourself

4-step recipe:
1. Define AgentTool that clones parent context and runs a nested query loop
2. Share system prompt + message history prefix for cache efficiency
3. Add SendMessage for inter-agent communication via mailbox pattern
4. Implement coordinator mode by restricting coordinator to Agent + SendMessage only

---

# Chapter 8: MCP Integration

## 8.1 Transport Types

```typescript
type McpTransport = 'stdio' | 'sse' | 'http' | 'ws' | 'sdk'

// stdio:  Local subprocess (most common for local tools)
// sse:    HTTP Server-Sent Events (remote servers)
// http:   Streamable HTTP (newer protocol)
// ws:     WebSocket (bidirectional)
// sdk:    In-process SDK (no network)
```

## 8.2 Connection Lifecycle

ASCII diagram:
```
Settings (5 config scopes) → Config Discovery
    ↓
connectToServer(config)
    ↓
Transport selection:
    stdio → spawn subprocess, StdioClientTransport
    sse   → SSEClientTransport with OAuth
    http  → StreamableHTTPClientTransport
    ↓
client.listTools() → MCPTool wrapper → Tool interface
    ↓
assembleToolPool() merges MCP tools with built-in tools
    ↓
Tool calls: mcp__server__toolName → route to client.callTool()
```

## 8.3 Reconnection Strategy

```typescript
const RECONNECT = {
  maxAttempts: 5,
  initialBackoffMs: 1000,
  maxBackoffMs: 30000,     // 30 seconds
  giveUpMs: 600000,        // 10 minutes
}

// Exponential backoff: delay = min(initialMs * 2^attempt, maxMs)
// On reconnect: re-fetch tools (ToolListChangedNotification)
// On auth failure: transition to 'needs-auth' state
```

## 8.4 Connection States

```typescript
type MCPServerConnection =
  | { type: 'connected',  client: Client, tools: Tool[] }
  | { type: 'failed',     error: string }
  | { type: 'needs-auth', serverName: string }
  | { type: 'pending',    reconnectAttempt: number }
  | { type: 'disabled',   reason: string }
```

## 8.5 Plugin MCP Dedup

```
Plugin servers namespaced: plugin:name:server
Content-based signature: hash(URL + command_hash)
Manual configs always win over plugin duplicates
First-loaded plugin wins among plugin servers
```

## 8.6 Build It Yourself

3-step recipe:
1. Create an MCP client with transport abstraction (start with stdio)
2. Fetch tools from server, wrap each as your Tool interface
3. Merge into assembleToolPool() with built-in tools
4. Add reconnection with exponential backoff

---

# Chapter 9: Slash Command System

## 9.1 Command Types

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)

// PromptCommand: Sends prompt to Claude API with restricted tool access
//   Example: /commit — AI generates commit message, only git tools allowed
//   Returns: ContentBlockParam[] (prompt for the model)

// LocalCommand: Runs JavaScript locally, no API call
//   Example: /cost — reads session cost, returns text
//   Returns: { type: 'text', value: string } | { type: 'compact' } | 'skip'

// LocalJSXCommand: Renders interactive React/Ink UI
//   Example: /config — opens settings panel
//   Returns: JSX element with onDone callback
```

## 9.2 Command Registry

```typescript
// All commands are lazy-loaded to keep startup fast
import commit from './commands/commit/index.js'
import review from './commands/review/index.js'

// Feature-gated conditional imports
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// Properties that control command behavior:
// isEnabled()              — runtime check (auth state, env vars)
// isHidden                 — hide from typeahead but still loadable
// disableModelInvocation   — model can't auto-invoke this command
// userInvocable            — available as /command in REPL
// immediate                — execute without waiting for stop point
// loadedFrom               — 'skills' | 'plugin' | 'managed' | 'bundled'
```

## 9.3 Skill Loading

```typescript
type LoadedFrom = 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'

// 5 directory sources (priority order):
// 1. Managed   → .claude/settings-managed/skills (org-pushed)
// 2. Project   → .claude/skills/ (repo-local)
// 3. User      → ~/.claude/skills/ (global)
// 4. Plugin    → from installed plugins
// 5. Bundled   → compiled into the binary

// Skills are Markdown files with frontmatter:
// ---
// name: my-skill
// description: What this skill does
// model: claude-sonnet-4-6
// allowed_tools: [Bash, Read, Edit]
// ---
// Prompt content here...

// Two execution modes:
// Inline → Expand prompt into current conversation context
// Fork   → Spawn isolated sub-agent with skill prompt
```

## 9.4 Build It Yourself

3-step recipe:
1. Define a Command interface with name, type, and execute/load methods
2. Register commands with lazy loading (import only when invoked)
3. Add skill directory scanning with frontmatter parsing

---

# Chapter 10: Terminal UI (React/Ink)

## 10.1 Why React in a Terminal

Ink renders React components to ANSI terminal output. Benefits:
- Component composition (reuse MessageView, ProgressBar, DiffViewer)
- Reactive updates (state change → re-render)
- Hooks (useState, useEffect, custom hooks like useCanUseTool)

## 10.2 Component Hierarchy

```
<AppStateProvider store={store}>
  <REPL>
    ├── <MessageList>
    │     ├── <UserMessage />
    │     ├── <AssistantMessage />
    │     ├── <ToolUseMessage>
    │     │     └── tool.renderToolUseMessage(input)
    │     ├── <ToolResultMessage>
    │     │     └── tool.renderToolResultMessage(output)
    │     └── <ToolProgressMessage>
    │           └── tool.renderToolUseProgressMessage(progress)
    ├── <PermissionDialog />
    ├── <TaskPanel />
    ├── <StatusLine />
    └── <PromptInput />
  </REPL>
</AppStateProvider>
```

## 10.3 Per-Tool Rendering

Each tool implements its own UI:
```typescript
// In GlobTool:
renderToolUseMessage(input) {
  return <Text>Finding files matching {input.pattern}...</Text>
}

renderToolResultMessage(output) {
  return <Box flexDirection="column">
    {output.filenames.map(f => <Text key={f}>{f}</Text>)}
    <Text dimColor>Found {output.numFiles} files in {output.durationMs}ms</Text>
  </Box>
}

// BashTool renders streaming terminal output
// FileEditTool renders colored diffs
// AgentTool renders progress bars for sub-agents
```

## 10.4 Build It Yourself

3-step recipe:
1. Use Ink (React for terminals) or a simpler TUI framework (blessed, bubbletea for Go)
2. Create a message renderer that dispatches to per-tool render functions
3. Use a reactive store (Zustand, Redux, or signals) for state → UI binding
4. Implement a permission dialog component for interactive approval

---

# Chapter 11: IDE Bridge Protocol

## 11.1 Architecture

```
IDE Extension (VS Code / JetBrains)
    ↕ WebSocket + SSE
Bridge Server (cloud)
    ↕ HTTP + WebSocket
CLI Process (local)
```

## 11.2 Key Types

```typescript
type BridgeConfig = {
  dir: string
  machineName: string
  branch: string
  maxSessions: number
  spawnMode: 'single-session' | 'worktree' | 'same-dir'
  bridgeId: string      // Client-generated UUID
  environmentId: string // For idempotent registration
}

type WorkSecret = {
  version: number
  session_ingress_token: string
  api_base_url: string
  auth: Array<{ type: string; token: string }>
  mcp_config?: unknown
  environment_variables?: Record<string, string>
}

type SessionHandle = {
  sessionId: string
  done: Promise<SessionDoneStatus>
  kill(): void
  writeStdin(data: string): void
}
```

## 11.3 Spawn Modes

```
single-session  → One CLI process per bridge connection
worktree        → Git worktree per session (full isolation)
same-dir        → Multiple sessions in same working directory
```

## 11.4 Permission Proxying

```
CLI needs permission → bridge sends control_request to IDE
IDE shows dialog → user decides → bridge sends control_response
CLI receives decision → continues tool execution
```

## 11.5 Backoff Configuration

```typescript
const BACKOFF = {
  connInitialMs: 2000,
  connCapMs: 120000,    // 2 minutes
  connGiveUpMs: 600000, // 10 minutes
  generalInitialMs: 500,
  generalCapMs: 30000,
  generalGiveUpMs: 600000,
}
```

## 11.6 Build It Yourself

3-step recipe:
1. Define a message protocol (JSON over WebSocket) for session control
2. Implement spawn modes (start with single-session)
3. Add permission proxying (forward CLI permission requests to IDE UI)

---

# Chapter 12: Memory & Cost Tracking

## 12.1 Memory System (MEMORY.md)

```typescript
const ENTRYPOINT_NAME = 'MEMORY.md'
const MAX_ENTRYPOINT_LINES = 200
const MAX_ENTRYPOINT_BYTES = 25_000  // ~125 chars/line

function truncateEntrypointContent(raw: string) {
  const lines = raw.trim().split('\n')

  // Dual-cap: lines first (natural boundary), then bytes
  let truncated = lines.length > MAX_LINES
    ? lines.slice(0, MAX_LINES).join('\n')
    : raw.trim()

  if (truncated.length > MAX_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_BYTES)
  }

  if (wasLineTruncated || wasByteTruncated) {
    truncated += '\n\n> WARNING: MEMORY.md truncated. Keep index entries short.'
  }
  return truncated
}
```

Memory types (stored as individual files with frontmatter):
```markdown
---
name: user-preferences
description: How the user likes to work
type: user | feedback | project | reference
---
Content here...
```

MEMORY.md is an index (one line per entry, <150 chars each), pointing to topic files.

## 12.2 Cost Tracking

```typescript
type StoredCostState = {
  totalCostUSD: number
  totalAPIDuration: number
  totalToolDuration: number
  totalLinesAdded: number
  totalLinesRemoved: number
  modelUsage: {
    [model: string]: {
      inputTokens: number
      outputTokens: number
      cacheReadInputTokens: number
      cacheCreationInputTokens: number
      costUSD: number
    }
  }
}

// Per-API-call: accumulate usage from response
function addToTotalSessionCost(cost, usage, model) {
  addToTotalModelUsage(cost, usage, model)
  costCounter?.add(cost, { model })
  tokenCounter?.add(usage.input_tokens, { model, type: 'input' })
  tokenCounter?.add(usage.output_tokens, { model, type: 'output' })
  tokenCounter?.add(usage.cache_read_input_tokens, { model, type: 'cacheRead' })
}

// Saved per-session to project config
function saveCurrentSessionCosts() {
  saveProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastModelUsage: getModelUsage(),
    lastSessionId: getSessionId(),
  }))
}
```

## 12.3 Build It Yourself

3-step recipe:
1. Create a MEMORY.md file with truncation (200 lines / 25KB dual cap)
2. Track per-model token usage (input, output, cache read, cache creation)
3. Calculate USD cost per model using published pricing tables
4. Persist cost state per-session to project config

---

# Appendix

## A.1 File Index

| System | Key File | Purpose |
|--------|----------|---------|
| Tool types | `src/Tool.ts` | Core Tool interface, buildTool(), ToolPermissionContext |
| Tool registry | `src/tools.ts` | getAllBaseTools(), getTools(), assembleToolPool() |
| Tool execution | `src/services/tools/toolExecution.ts` | runToolUse(), 6-phase pipeline |
| Streaming executor | `src/services/tools/StreamingToolExecutor.ts` | Parallel tool execution |
| Query engine | `src/QueryEngine.ts` | submitMessage(), session state |
| Query loop | `src/query.ts` | queryLoop(), recovery, budget gates |
| Bootstrap | `src/main.tsx` | Startup prefetch, CLI parsing |
| Setup | `src/setup.ts` | Initialization sequence |
| Context | `src/context.ts` | System prompt assembly |
| Permissions | `src/utils/permissions/` | Rule checking, classifiers |
| State | `src/state/AppStateStore.ts` | AppState type, store |
| Compact | `src/services/compact/compact.ts` | Context compression |
| Coordinator | `src/coordinator/coordinatorMode.ts` | Multi-agent system prompt |
| Forked agents | `src/utils/forkedAgent.ts` | CacheSafeParams, cache sharing |
| MCP client | `src/services/mcp/client.ts` | Connection management |
| MCP types | `src/services/mcp/types.ts` | Transport, config schemas |
| Skills | `src/skills/loadSkillsDir.ts` | Skill directory scanning |
| Commands | `src/commands.ts` | Command registry |
| Bridge | `src/bridge/types.ts` | IDE bridge protocol |
| Bridge main | `src/bridge/bridgeMain.ts` | Bridge orchestration |
| Memory | `src/memdir/memdir.ts` | MEMORY.md management |
| Cost | `src/cost-tracker.ts` | Token/USD tracking |
| GlobTool | `src/tools/GlobTool/GlobTool.ts` | Reference tool implementation |
| BashTool | `src/tools/BashTool/BashTool.tsx` | Complex tool with streaming |
| AgentTool | `src/tools/AgentTool/AgentTool.tsx` | Sub-agent spawning |

## A.2 Glossary

| Term | Definition |
|------|-----------|
| **ToolUseContext** | Runtime context passed to every tool call -- includes tools list, app state, abort controller, file cache, MCP clients, and permission context |
| **AppState** | Immutable application state (Zustand-like store) -- settings, model, permissions, MCP, plugins, tasks |
| **CacheSafeParams** | Parameters that must match between parent and forked agent to share prompt cache -- system prompt, user context, tools, messages |
| **ToolPermissionContext** | Frozen permission rules from all sources (policy, project, user, CLI, session) with deny/allow/ask behaviors |
| **StreamingToolExecutor** | Executes tools as they stream in from the API, managing concurrency (safe tools run in parallel, unsafe tools run exclusively) |
| **Reactive compact** | Automatic context compression triggered when messages exceed token threshold -- summarizes old messages, restores key files |
| **MCPTool** | Wrapper that adapts an MCP server's tool to the internal Tool interface -- passthrough input/output, 100K char limit |
| **Forked agent** | Sub-agent that shares the parent's prompt cache via CacheSafeParams -- used for skills, post-turn analysis, and background work |
| **Prompt cache** | Anthropic API feature where identical system prompt + message prefix = cached (cheaper, faster). Cache key includes: system prompt, tools, model, messages, thinking config |
| **Trust gate** | Deferred initialization that only runs after workspace trust is established -- plugins, skills, MCP, hooks are blocked until trusted |

## A.3 Further Reading

- Claude API Documentation: https://docs.anthropic.com/en/docs
- MCP Specification: https://modelcontextprotocol.io
- Ink (React for terminals): https://github.com/vadimdemedes/ink
- Zod (TypeScript schema validation): https://zod.dev
