# Chapter 8: MCP Integration

MCP turns the harness into a broader tool and context platform. This chapter covers treating remote capabilities as runtime-managed resources — with real transport selection, reconnection, and deduplication code.

## Why this system exists

MCP expands what the model can reach, but it also expands the failure surface. The harness needs a clean integration layer that handles connection lifecycle, tool deduplication, and graceful degradation before exposing remote tools to the model.

## Transport types

MCP supports multiple transports. Your adapter picks one based on server config:

```python
from enum import Enum

class McpTransport(Enum):
    STDIO = "stdio"      # local subprocess (most common)
    SSE = "sse"          # HTTP Server-Sent Events (remote)
    HTTP = "http"        # streamable HTTP (newer protocol)
    WEBSOCKET = "ws"     # bidirectional WebSocket
    SDK = "sdk"          # in-process (no network)
```

## Connection state machine

```python
from dataclasses import dataclass, field

class ConnectionStatus(Enum):
    CONNECTED = "connected"
    FAILED = "failed"
    NEEDS_AUTH = "needs_auth"
    PENDING = "pending"       # reconnecting with backoff
    DISABLED = "disabled"

@dataclass(frozen=True)
class ServerConnection:
    name: str
    transport: McpTransport
    status: ConnectionStatus
    tools: tuple[dict, ...] = ()
    error: str | None = None
    reconnect_attempt: int = 0
```

## Python MCP client adapter

```python
import asyncio
import json
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class McpServerConfig:
    name: str
    transport: McpTransport
    command: str | None = None       # for stdio
    url: str | None = None           # for sse/http/ws
    env: dict[str, str] = field(default_factory=dict)

class McpClientAdapter:
    """Manages MCP server connections and converts tools to internal format."""

    def __init__(self):
        self._connections: dict[str, ServerConnection] = {}

    async def connect(self, config: McpServerConfig) -> ServerConnection:
        """Connect to an MCP server and discover its tools."""
        try:
            if config.transport == McpTransport.STDIO:
                tools = await self._connect_stdio(config)
            elif config.transport in (McpTransport.SSE, McpTransport.HTTP):
                tools = await self._connect_remote(config)
            else:
                tools = ()

            conn = ServerConnection(
                name=config.name,
                transport=config.transport,
                status=ConnectionStatus.CONNECTED,
                tools=tools,
            )
        except PermissionError:
            conn = ServerConnection(
                name=config.name,
                transport=config.transport,
                status=ConnectionStatus.NEEDS_AUTH,
            )
        except Exception as exc:
            conn = ServerConnection(
                name=config.name,
                transport=config.transport,
                status=ConnectionStatus.FAILED,
                error=str(exc),
            )

        self._connections[config.name] = conn
        return conn

    async def _connect_stdio(self, config: McpServerConfig) -> tuple[dict, ...]:
        """Spawn subprocess and discover tools via stdio JSON-RPC."""
        proc = await asyncio.create_subprocess_exec(
            *config.command.split(),
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            env={**config.env},
        )
        # Send tools/list JSON-RPC request
        request = json.dumps({"jsonrpc": "2.0", "id": 1, "method": "tools/list"})
        proc.stdin.write(f"{request}\n".encode())
        await proc.stdin.drain()

        line = await proc.stdout.readline()
        response = json.loads(line)
        return tuple(response.get("result", {}).get("tools", []))

    async def _connect_remote(self, config: McpServerConfig) -> tuple[dict, ...]:
        """Connect via HTTP/SSE and discover tools."""
        import httpx
        async with httpx.AsyncClient() as http:
            resp = await http.post(
                f"{config.url}/mcp/v1",
                json={"jsonrpc": "2.0", "id": 1, "method": "tools/list"},
            )
            data = resp.json()
            return tuple(data.get("result", {}).get("tools", []))

    def get_all_tools(self) -> list[dict]:
        """Return tools from all connected servers, prefixed with server name."""
        tools = []
        for conn in self._connections.values():
            if conn.status != ConnectionStatus.CONNECTED:
                continue
            for tool in conn.tools:
                tools.append({
                    **tool,
                    "name": f"mcp__{conn.name}__{tool['name']}",
                })
        return tools
```

## Reconnection with exponential backoff

```python
@dataclass(frozen=True)
class BackoffConfig:
    initial_ms: int = 1_000
    max_ms: int = 30_000
    give_up_ms: int = 600_000   # 10 minutes

async def reconnect_with_backoff(
    adapter: McpClientAdapter,
    config: McpServerConfig,
    backoff: BackoffConfig = BackoffConfig(),
) -> ServerConnection:
    """Reconnect to a failed server with exponential backoff."""
    import time
    start = time.monotonic()
    attempt = 0

    while True:
        delay_ms = min(backoff.initial_ms * (2 ** attempt), backoff.max_ms)
        elapsed_ms = (time.monotonic() - start) * 1000

        if elapsed_ms >= backoff.give_up_ms:
            return ServerConnection(
                name=config.name,
                transport=config.transport,
                status=ConnectionStatus.DISABLED,
                error=f"gave up after {elapsed_ms:.0f}ms",
            )

        await asyncio.sleep(delay_ms / 1000)
        attempt += 1

        conn = await adapter.connect(config)
        if conn.status == ConnectionStatus.CONNECTED:
            return conn
```

## Tool deduplication

When built-in tools and MCP tools share a name, built-in wins:

```python
def deduplicate_tools(
    builtin_tools: list[dict],
    mcp_tools: list[dict],
) -> list[dict]:
    """Merge built-in and MCP tools. Built-in wins on name conflict."""
    seen: set[str] = set()
    result: list[dict] = []

    # Built-ins first (stable cache prefix)
    for tool in sorted(builtin_tools, key=lambda t: t["name"]):
        if tool["name"] not in seen:
            seen.add(tool["name"])
            result.append(tool)

    # MCP tools second
    for tool in sorted(mcp_tools, key=lambda t: t["name"]):
        if tool["name"] not in seen:
            seen.add(tool["name"])
            result.append(tool)

    return result
```

## Plugin MCP dedup

When multiple plugins provide the same MCP server, deduplicate by content signature:

```python
import hashlib

def server_signature(config: McpServerConfig) -> str:
    """Content-based signature for dedup. Manual configs always win."""
    raw = f"{config.transport.value}:{config.command or config.url}"
    return hashlib.sha256(raw.encode()).hexdigest()[:16]

def deduplicate_plugin_servers(
    plugin_servers: list[McpServerConfig],
    manual_servers: list[McpServerConfig],
) -> list[McpServerConfig]:
    """Manual configs win. Among plugins, first-loaded wins."""
    seen = {server_signature(s) for s in manual_servers}
    result = list(manual_servers)

    for server in plugin_servers:
        sig = server_signature(server)
        if sig not in seen:
            seen.add(sig)
            result.append(server)

    return result
```

## Failure modes and tradeoffs

| Failure | Wrong approach | Right approach |
|---------|---------------|----------------|
| Server disconnects | Crash the harness | Set status to `PENDING`, reconnect with backoff |
| Duplicate tool names | Expose both (model confusion) | Deduplicate — built-in wins |
| Auth expired | Retry forever | Set `NEEDS_AUTH`, surface to user |
| Slow server | Block the loop | Timeout per-call, degrade gracefully |

## Build-it-yourself checklist

- [ ] Write an MCP adapter that converts server tools to your internal registry format
- [ ] Track per-server connection state (`connected` / `failed` / `needs_auth` / `pending`)
- [ ] Implement reconnection with exponential backoff (1s → 30s cap, 10min give-up)
- [ ] Deduplicate tools before exposing to the model (built-in wins on conflict)
- [ ] Namespace MCP tools: `mcp__{server}__{tool}` to avoid collisions
- [ ] Keep your permission policy in front of remote execution
- [ ] Define degraded behavior when a server is unavailable

## Reference provenance

- [Open this chapter inside the full blueprint](../full-blueprint.md#chapter-8-mcp-integration)
- Patterns extracted from: transport types, connection lifecycle, reconnect strategy, tool deduplication, plugin server dedup
