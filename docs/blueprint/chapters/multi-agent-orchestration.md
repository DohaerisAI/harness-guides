# Chapter 7: Multi-Agent Orchestration

Complex tasks benefit from delegation. A coordinator breaks work into bounded subtasks and assigns them to workers. The coordinator retains control, synthesizes results, and communicates with the user. Without this separation, multi-agent systems become expensive confusion amplifiers.

## Architecture

```
┌──────────────────────────────────────┐
│ Coordinator                          │
│   Tools: Agent, SendMessage, TaskStop│
│   Role: plan, delegate, synthesize   │
├──────────────────────────────────────┤
│   ↓ spawn       ↓ spawn      ↓ spawn│
│ Worker 1      Worker 2     Worker 3  │
│ (full tools) (full tools) (full tools)│
│ bash, read,  bash, read,  bash, read,│
│ edit, grep   edit, grep   edit, grep │
└──────────────────────────────────────┘

Communication: task-notification XML
Coordinator ←── <task-notification> ←── Worker
Coordinator ──→ SendMessage ──→ Worker
```

The coordinator never touches files directly. It only spawns workers, sends them messages, and stops them. Workers get the real tools --- bash, read, edit, grep --- and report results back as structured XML.

## The Coordinator System Prompt

This is the actual prompt from Claude Code that teaches the coordinator how to manage workers.

??? note "Full Coordinator System Prompt (from Claude Code source)"

    ```
    You are Claude Code, an AI assistant that orchestrates software engineering tasks
    across multiple workers.

    ## 1. Your Role
    You are a coordinator. Your job is to:
    - Help the user achieve their goal
    - Direct workers to research, implement and verify code changes
    - Synthesize results and communicate with the user
    - Answer questions directly when possible

    Every message you send is to the user. Worker results are internal signals —
    never thank them.

    ## 2. Your Tools
    - Agent — Spawn a new worker
    - SendMessage — Continue an existing worker
    - TaskStop — Stop a running worker

    When calling Agent:
    - Don't use one worker to check on another
    - Don't set the model parameter
    - Continue workers via SendMessage to reuse their context
    - After launching agents, briefly tell the user what you launched and end
      your response

    ### Agent Results
    Worker results arrive as user-role messages containing <task-notification> XML:
    <task-notification>
    <task-id>{agentId}</task-id>
    <status>completed|failed|killed</status>
    <summary>{status summary}</summary>
    <result>{agent's response}</result>
    <usage>
      <total_tokens>N</total_tokens>
      <tool_uses>N</tool_uses>
      <duration_ms>N</duration_ms>
    </usage>
    </task-notification>

    ## 3. Task Workflow
    | Phase          | Who                | Purpose                      |
    |----------------|--------------------|------------------------------|
    | Research        | Workers (parallel) | Investigate codebase         |
    | Synthesis       | Coordinator        | Read findings, craft specs   |
    | Implementation  | Workers            | Make changes per spec        |
    | Verification    | Workers            | Test changes work            |

    Parallelism is your superpower. Launch independent workers concurrently.

    ## 4. Writing Worker Prompts
    Workers can't see your conversation. Every prompt must be self-contained.
    Always synthesize research findings before delegating implementation.

    Good: "Fix null pointer in src/auth/validate.ts:42. Add null check before
           user.id access. The function signature is validate(user: User) and
           user can be undefined when session expires."
    Bad:  "Based on your findings, fix the bug."

    ## 5. Continue vs Spawn
    | Situation                              | Mechanism    | Why                        |
    |----------------------------------------|------------- |----------------------------|
    | Research explored the files to edit     | Continue     | Worker has files in context |
    | Research was broad, implementation narrow| Spawn fresh | Clean context              |
    | Correcting a failure                   | Continue     | Worker has error context    |
    | Verifying another worker's code        | Spawn fresh  | Fresh eyes                 |
    ```

Three things to notice. First, the coordinator never gets file-editing tools --- it can only talk to workers and to the user. Second, worker results arrive as `<task-notification>` XML injected into the coordinator's conversation as user-role messages. Third, the prompt explicitly tells the coordinator to synthesize research before delegating implementation. Without that rule, workers get vague instructions and produce vague output.

## Python implementation

### Worker task model

Represent a delegated job as an immutable value object. Workers get a self-contained prompt, a bounded tool set, and nothing else.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class WorkerTask:
    """A bounded unit of work delegated to a worker agent."""
    task_id: str
    description: str
    prompt: str
    allowed_tools: tuple[str, ...] = ()
    model: str | None = None
```

### Task notification

Workers report results as structured XML. Parse it on the coordinator side.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class TaskNotification:
    """Structured result from a completed worker."""
    task_id: str
    status: str          # "completed" | "failed" | "killed"
    summary: str
    result: str
    total_tokens: int
    tool_uses: int
    duration_ms: int

    def to_xml(self) -> str:
        return (
            f"<task-notification>\n"
            f"<task-id>{self.task_id}</task-id>\n"
            f"<status>{self.status}</status>\n"
            f"<summary>{self.summary}</summary>\n"
            f"<result>{self.result}</result>\n"
            f"<usage>"
            f"<total_tokens>{self.total_tokens}</total_tokens>"
            f"<tool_uses>{self.tool_uses}</tool_uses>"
            f"<duration_ms>{self.duration_ms}</duration_ms>"
            f"</usage>\n"
            f"</task-notification>"
        )
```

### Coordinator tool definitions

The coordinator only gets three tools. Everything else belongs to workers.

```python
COORDINATOR_TOOLS = [
    {
        "name": "spawn_worker",
        "description": "Spawn a new worker agent with a self-contained task.",
        "input_schema": {
            "type": "object",
            "properties": {
                "description": {
                    "type": "string",
                    "description": "Brief label for this worker's task.",
                },
                "prompt": {
                    "type": "string",
                    "description": "Complete, self-contained instructions for the worker.",
                },
            },
            "required": ["description", "prompt"],
        },
    },
    {
        "name": "send_message",
        "description": "Send a follow-up message to an existing worker.",
        "input_schema": {
            "type": "object",
            "properties": {
                "task_id": {"type": "string"},
                "message": {"type": "string"},
            },
            "required": ["task_id", "message"],
        },
    },
    {
        "name": "stop_worker",
        "description": "Stop a running worker.",
        "input_schema": {
            "type": "object",
            "properties": {
                "task_id": {"type": "string"},
            },
            "required": ["task_id"],
        },
    },
]
```

### Coordinator class

=== "Anthropic"

    ```python
    import anthropic
    import uuid
    import asyncio
    from dataclasses import dataclass, field

    COORDINATOR_SYSTEM = """You are a coordinator. You break tasks into bounded
    subtasks and delegate them to workers using spawn_worker. You never edit files
    directly. Synthesize worker results and communicate with the user."""

    WORKER_SYSTEM = """You are a worker agent. Complete the assigned task using
    the tools available to you. Report your findings clearly and concisely."""


    @dataclass
    class Coordinator:
        client: anthropic.Anthropic
        system_prompt: str = COORDINATOR_SYSTEM
        model: str = "claude-sonnet-4-20250514"
        workers: dict[str, WorkerTask] = field(default_factory=dict)
        messages: list[dict] = field(default_factory=list)

        async def run(self, user_input: str) -> str:
            self.messages = [
                *self.messages,
                {"role": "user", "content": user_input},
            ]

            while True:
                response = self.client.messages.create(
                    model=self.model,
                    max_tokens=8096,
                    system=self.system_prompt,
                    messages=self.messages,
                    tools=COORDINATOR_TOOLS,
                )
                self.messages = [
                    *self.messages,
                    {"role": "assistant", "content": response.content},
                ]

                tool_blocks = [b for b in response.content if b.type == "tool_use"]
                if not tool_blocks:
                    # Extract final text response
                    return next(
                        (b.text for b in response.content if hasattr(b, "text")),
                        "",
                    )

                # Run independent workers in parallel
                results = await asyncio.gather(
                    *(self._handle_tool(block) for block in tool_blocks)
                )

                self.messages = [
                    *self.messages,
                    {"role": "user", "content": list(results)},
                ]

        async def _handle_tool(self, block) -> dict:
            if block.name == "spawn_worker":
                return await self._spawn_worker(block)
            if block.name == "send_message":
                return self._send_message(block)
            if block.name == "stop_worker":
                return self._stop_worker(block)
            return {
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": f"Unknown tool: {block.name}",
                "is_error": True,
            }

        async def _spawn_worker(self, block) -> dict:
            task_id = f"agent-{uuid.uuid4().hex[:8]}"
            task = WorkerTask(
                task_id=task_id,
                description=block.input["description"],
                prompt=block.input["prompt"],
            )
            self.workers = {**self.workers, task_id: task}

            worker_result = await self._run_worker(task)
            notification = TaskNotification(
                task_id=task_id,
                status="completed",
                summary=task.description,
                result=worker_result,
                total_tokens=0,
                tool_uses=0,
                duration_ms=0,
            )
            return {
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": notification.to_xml(),
            }

        async def _run_worker(self, task: WorkerTask) -> str:
            """Run a worker as a separate, isolated conversation."""
            response = self.client.messages.create(
                model=task.model or self.model,
                max_tokens=8096,
                system=WORKER_SYSTEM,
                messages=[{"role": "user", "content": task.prompt}],
            )
            return next(
                (b.text for b in response.content if hasattr(b, "text")),
                "",
            )

        def _send_message(self, block) -> dict:
            task_id = block.input["task_id"]
            if task_id not in self.workers:
                return {
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": f"No worker found with id: {task_id}",
                    "is_error": True,
                }
            return {
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": f"Message sent to {task_id}",
            }

        def _stop_worker(self, block) -> dict:
            task_id = block.input["task_id"]
            self.workers = {
                k: v for k, v in self.workers.items() if k != task_id
            }
            return {
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": f"Worker {task_id} stopped",
            }
    ```

=== "OpenAI"

    ```python
    from openai import OpenAI
    import uuid
    import json
    import asyncio
    from dataclasses import dataclass, field

    OPENAI_TOOLS = [
        {
            "type": "function",
            "function": {
                "name": "spawn_worker",
                "description": "Spawn a new worker agent with a self-contained task.",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "description": {"type": "string"},
                        "prompt": {"type": "string"},
                    },
                    "required": ["description", "prompt"],
                },
            },
        },
        {
            "type": "function",
            "function": {
                "name": "send_message",
                "description": "Send a follow-up message to an existing worker.",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "task_id": {"type": "string"},
                        "message": {"type": "string"},
                    },
                    "required": ["task_id", "message"],
                },
            },
        },
        {
            "type": "function",
            "function": {
                "name": "stop_worker",
                "description": "Stop a running worker.",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "task_id": {"type": "string"},
                    },
                    "required": ["task_id"],
                },
            },
        },
    ]

    COORDINATOR_INSTRUCTIONS = """You are a coordinator. You break tasks into
    bounded subtasks and delegate them to workers using spawn_worker. You never
    edit files directly. Synthesize worker results and communicate with the user."""


    @dataclass
    class Coordinator:
        client: OpenAI
        instructions: str = COORDINATOR_INSTRUCTIONS
        model: str = "gpt-4.1"
        workers: dict[str, WorkerTask] = field(default_factory=dict)
        previous_response_id: str | None = None

        async def run(self, user_input: str) -> str:
            current_input = user_input

            while True:
                response = self.client.responses.create(
                    model=self.model,
                    instructions=self.instructions,
                    input=current_input,
                    tools=OPENAI_TOOLS,
                    previous_response_id=self.previous_response_id,
                )
                self.previous_response_id = response.id

                function_calls = [
                    item for item in response.output
                    if item.type == "function_call"
                ]
                if not function_calls:
                    return next(
                        (
                            item.content
                            for item in response.output
                            if item.type == "message"
                        ),
                        "",
                    )

                results = await asyncio.gather(
                    *(self._handle_call(call) for call in function_calls)
                )
                current_input = list(results)

        async def _handle_call(self, call) -> dict:
            args = json.loads(call.arguments)

            if call.name == "spawn_worker":
                task_id = f"agent-{uuid.uuid4().hex[:8]}"
                task = WorkerTask(
                    task_id=task_id,
                    description=args["description"],
                    prompt=args["prompt"],
                )
                self.workers = {**self.workers, task_id: task}
                worker_result = await self._run_worker(task)
                notification = TaskNotification(
                    task_id=task_id,
                    status="completed",
                    summary=task.description,
                    result=worker_result,
                    total_tokens=0,
                    tool_uses=0,
                    duration_ms=0,
                )
                return {
                    "type": "function_call_output",
                    "call_id": call.call_id,
                    "output": notification.to_xml(),
                }

            if call.name == "send_message":
                return {
                    "type": "function_call_output",
                    "call_id": call.call_id,
                    "output": f"Message sent to {args['task_id']}",
                }

            if call.name == "stop_worker":
                self.workers = {
                    k: v for k, v in self.workers.items()
                    if k != args["task_id"]
                }
                return {
                    "type": "function_call_output",
                    "call_id": call.call_id,
                    "output": f"Worker {args['task_id']} stopped",
                }

            return {
                "type": "function_call_output",
                "call_id": call.call_id,
                "output": f"Unknown tool: {call.name}",
            }

        async def _run_worker(self, task: WorkerTask) -> str:
            """Run a worker as a separate conversation."""
            response = self.client.responses.create(
                model=self.model,
                instructions="You are a worker agent. Complete the task.",
                input=task.prompt,
            )
            return next(
                (
                    item.content
                    for item in response.output
                    if item.type == "message"
                ),
                "",
            )
    ```

## Prompt cache sharing

When spawning sub-agents, share the parent's prompt cache to save money. The Anthropic API caches the system prompt and tools prefix. If every worker uses the same system prompt, tools list, and model, the cached portion costs 10x less on cache hits.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class CacheSafeParams:
    """Parameters that must match the parent for prompt cache hits."""
    system_prompt: str          # Must be identical across workers
    tools: tuple[dict, ...]     # Same tools in same order
    model: str                  # Same model string
    context_prefix: tuple       # Parent's message prefix (if sharing)
```

In practice this means: define your worker system prompt and tool list once, then reuse them for every worker spawn. Do not customize per-worker unless the task genuinely requires different tools.

```python
# Shared config reused across all workers
SHARED_WORKER_CONFIG = CacheSafeParams(
    system_prompt=WORKER_SYSTEM,
    tools=tuple(COORDINATOR_TOOLS),
    model="claude-sonnet-4-20250514",
    context_prefix=(),
)

async def spawn_with_cache(
    client: anthropic.Anthropic,
    config: CacheSafeParams,
    prompt: str,
) -> str:
    """Spawn a worker using shared config for cache efficiency."""
    response = client.messages.create(
        model=config.model,
        max_tokens=8096,
        system=config.system_prompt,
        messages=[{"role": "user", "content": prompt}],
    )
    return next(
        (b.text for b in response.content if hasattr(b, "text")),
        "",
    )
```

## SendMessage routing

Workers and coordinators can communicate through multiple transports. The coordinator routes messages based on the target identifier prefix.

```python
def route_message(
    to: str,
    message: str,
    workers: dict[str, WorkerTask],
) -> str:
    """Route a message to a worker by target identifier."""
    if to == "*":
        # Broadcast to all active workers
        for worker in workers.values():
            _deliver(worker.task_id, message)
        return f"Broadcast to {len(workers)} workers"

    if to.startswith("uds:"):
        # Unix domain socket peer
        return _send_to_uds(to[4:], message)

    if to.startswith("bridge:"):
        # IDE bridge session
        return _send_to_bridge(to[7:], message)

    # Direct to named worker
    if to in workers:
        _deliver(to, message)
        return f"Delivered to {to}"

    return f"No worker found: {to}"


def _deliver(task_id: str, message: str) -> None:
    """Deliver a message to a worker's inbox (implementation-specific)."""
    pass

def _send_to_uds(path: str, message: str) -> str:
    """Send via Unix domain socket."""
    return f"Sent to UDS: {path}"

def _send_to_bridge(session: str, message: str) -> str:
    """Send via IDE bridge."""
    return f"Sent to bridge: {session}"
```

## Build it yourself

1. Create a coordinator that only has `spawn_worker`, `send_message`, and `stop_worker` tools. No file access, no bash.
2. Workers get the full tool set --- bash, read, edit, grep --- and a self-contained prompt that describes exactly what to do.
3. Workers report results as `<task-notification>` XML. The coordinator parses the XML and decides what to do next.
4. The coordinator synthesizes research findings before delegating implementation. Never let a worker implement something based on another worker's raw output --- the coordinator must read, filter, and write a spec first.
5. Share system prompt and tools across workers for prompt cache efficiency. Every worker with the same `(system_prompt, tools, model)` tuple gets cache hits on the prefix.
6. Use `asyncio.gather` to run independent workers in parallel. Sequential workers waste time and money.

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-7-multi-agent-orchestration)
- The coordinator system prompt is derived from Claude Code's multi-agent orchestration mode
- Cache-sharing rules come from the Anthropic API prompt caching documentation
