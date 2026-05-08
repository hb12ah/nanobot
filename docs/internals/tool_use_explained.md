# Tool Use — How It Works

This document explains the full lifecycle of tool use in nanobot: how tools are defined, how the LLM learns about them, how it decides to call them, and how results flow back.

---

## The Big Picture

```
┌──────────────────────────────────────────────────────────────────┐
│                         AgentRunner.run()                        │
│                                                                  │
│  ┌─────────────┐    tool definitions    ┌──────────────────────┐ │
│  │ToolRegistry │ ──get_definitions()──► │   LLM API Request    │ │
│  │             │                        │  (messages + tools)  │ │
│  │ [Tool, ...] │                        └──────────┬───────────┘ │
│  └─────────────┘                                   │             │
│                                                    ▼             │
│                                        ┌──────────────────────┐  │
│                                        │    LLMResponse       │  │
│                                        │  finish_reason:      │  │
│                                        │    "tool_calls" ──►  │  │
│                                        │  tool_calls: [...]   │  │
│                                        └──────────┬───────────┘  │
│                                                   │              │
│                              should_execute_tools?│              │
│                                    YES ▼          │ NO           │
│                                                   │              │
│  ┌────────────────────────────────────┐           │ ▼            │
│  │       _execute_tools()             │     final_content        │
│  │                                    │       → break            │
│  │  for each ToolCallRequest:         │                          │
│  │    ToolRegistry.prepare_call()     │                          │
│  │      → cast + validate params      │                          │
│  │    tool.execute(**params)          │                          │
│  │    → append role="tool" message    │                          │
│  └────────────────┬───────────────────┘                          │
│                   │                                              │
│                   └──────── loop back to LLM call ──────────────┘
└──────────────────────────────────────────────────────────────────┘
```

---

## Step 1 — Tool Registration

Every tool is a Python class that subclasses `Tool` ([nanobot/agent/tools/base.py](../../nanobot/agent/tools/base.py)):

```python
@tool_parameters({
    "type": "object",
    "properties": {"path": {"type": "string"}},
    "required": ["path"],
})
class ReadFileTool(Tool):
    name = "read_file"
    description = "Read a file from disk."

    async def execute(self, *, path: str) -> str:
        ...
```

The `@tool_parameters` decorator attaches the JSON Schema and injects a concrete `parameters` property. Tools expose three things the system needs:

| Property / method | Purpose |
|---|---|
| `name` | Identifier used in function calls |
| `description` | Shown to the LLM so it knows what the tool does |
| `parameters` | JSON Schema for argument validation |
| `execute(**kwargs)` | The actual implementation |

Tools are registered into a `ToolRegistry` ([nanobot/agent/tools/registry.py](../../nanobot/agent/tools/registry.py)):

```python
registry.register(ReadFileTool())
```

---

## Step 2 — Tool Definitions Sent to the LLM

Before each LLM call, `AgentRunner._request_model()` asks the registry for all definitions:

```python
tools=spec.tools.get_definitions()  # runner.py:614
```

`ToolRegistry.get_definitions()` calls `tool.to_schema()` on every registered tool, which emits OpenAI-style function schemas:

```json
{
  "type": "function",
  "function": {
    "name": "read_file",
    "description": "Read a file from disk.",
    "parameters": { "type": "object", "properties": { "path": { "type": "string" } }, "required": ["path"] }
  }
}
```

Built-ins are sorted alphabetically first, then MCP tools — giving a stable ordering so Anthropic's prompt cache isn't invalidated on every call.

### Format conversion for Anthropic

`AnthropicProvider._convert_tools()` ([anthropic_provider.py:337](../../nanobot/providers/anthropic_provider.py#L337)) translates the OpenAI schema into Anthropic's format before the API call:

```json
{
  "name": "read_file",
  "description": "Read a file from disk.",
  "input_schema": { "type": "object", "properties": { "path": { "type": "string" } }, "required": ["path"] }
}
```

---

## Step 3 — The LLM Decides to Call a Tool

Claude sees the tool definitions in the API request and decides autonomously which tool(s) to call based on its reasoning. The decision is entirely the model's — nanobot controls only:

- **Which tools are available** (via `tools=` parameter)
- **Whether tool use is required** (via `tool_choice=`)

`_convert_tool_choice()` ([anthropic_provider.py:356](../../nanobot/providers/anthropic_provider.py#L356)) maps the abstract choice string to Anthropic's format:

| nanobot value | Anthropic format | Meaning |
|---|---|---|
| `"auto"` or `None` | `{"type": "auto"}` | Model decides |
| `"required"` | `{"type": "any"}` | Must call at least one tool |
| `"none"` | omitted | No tools available |
| `{"function": {"name": "X"}}` | `{"type": "tool", "name": "X"}` | Must call this specific tool |

When extended thinking is enabled, `tool_choice` is always forced to `auto` regardless of the spec.

---

## Step 4 — Parsing the LLM Response

When Claude calls a tool, its response contains `tool_use` blocks instead of (or in addition to) text:

```
stop_reason: "tool_use"
content:
  - {type: "text",     text: "Let me read that file."}
  - {type: "tool_use", id: "toolu_abc123", name: "read_file", input: {"path": "/tmp/foo"}}
```

`AnthropicProvider._parse_response()` ([anthropic_provider.py:485](../../nanobot/providers/anthropic_provider.py#L485)) iterates over the content blocks and separates them:

- `text` → accumulated into `content`
- `tool_use` → converted to `ToolCallRequest(id, name, arguments)`
- `thinking` → stored in `thinking_blocks`

The Anthropic `stop_reason` is mapped to a normalized `finish_reason`:

| Anthropic stop_reason | finish_reason |
|---|---|
| `"tool_use"` | `"tool_calls"` |
| `"end_turn"` | `"stop"` |
| `"max_tokens"` | `"length"` |

The result is a `LLMResponse` with a `tool_calls` list and `finish_reason = "tool_calls"`.

---

## Step 5 — Executing Tools

Back in `AgentRunner.run()`, the response is checked:

```python
if response.should_execute_tools:   # runner.py:285
```

`should_execute_tools` is `True` only when both conditions hold (from [base.py:73](../../nanobot/providers/base.py#L73)):

```python
return self.has_tool_calls and self.finish_reason in ("tool_calls", "stop")
```

This guard prevents executing tool calls that arrived under `refusal`, `content_filter`, or `error`.

### Execution flow

```
_execute_tools(tool_calls)
  └── _partition_tool_batches()          # concurrency_safe tools batched together
       └── for each batch:
            ├── (concurrent) asyncio.gather(*[_run_tool(...)])
            └── (sequential) for each call: _run_tool(...)

_run_tool(tool_call)
  ├── repeated_external_lookup_error()  # throttle identical external requests
  ├── ToolRegistry.prepare_call()
  │     ├── resolve tool by name
  │     ├── tool.cast_params()          # coerce string→int, etc.
  │     └── tool.validate_params()      # JSON Schema validation
  └── tool.execute(**params)            # the actual implementation
```

`_run_tool` is in [runner.py:742](../../nanobot/runner.py#L742). Concurrency batching is in [runner.py:1179](../../nanobot/runner.py#L1179).

### Concurrency

Tools declare their concurrency behavior on the `Tool` base class:

| Property | Meaning |
|---|---|
| `read_only = True` | Safe to parallelize (implies `concurrency_safe`) |
| `exclusive = True` | Must run alone even in concurrent mode |
| `concurrency_safe` | `read_only and not exclusive` |

When `spec.concurrent_tools` is enabled, `_partition_tool_batches` groups adjacent concurrency-safe tools and runs them with `asyncio.gather`. Non-safe tools run alone.

### Security boundaries inside tool execution

Before returning a tool result to the LLM, `_run_tool` checks for two policy violations:

- **SSRF** — private/internal URL detected → permanent non-retryable block with an explanation note appended.
- **Workspace violation** — path outside configured workspace → soft error with retry hint; escalates to a stronger hint on repeated attempts.

---

## Step 6 — Feeding Results Back

Each tool result is appended to the message history as a `role="tool"` message:

```python
{
  "role": "tool",
  "tool_call_id": "toolu_abc123",
  "name": "read_file",
  "content": "<file contents>",
}
```

`AnthropicProvider._convert_messages()` ([anthropic_provider.py:121](../../nanobot/providers/anthropic_provider.py#L121)) translates these into Anthropic's `tool_result` blocks inside a `user` turn when building the next request:

```json
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_abc123", "content": "..."}]}
```

The runner then loops back to Step 3 — the LLM now has the tool results in context and decides whether to call more tools or produce a final text response.

---

## Step 7 — Context Governance Before Each LLM Call

Before calling the LLM on each iteration, the runner applies several transformations to keep the message list well-formed ([runner.py:257–264](../../nanobot/runner.py#L257)):

| Pass | What it does |
|---|---|
| `_drop_orphan_tool_results` | Remove `role="tool"` messages with no matching assistant `tool_call` |
| `_backfill_missing_tool_results` | Insert synthetic error results for tool calls that have no result (e.g. interrupted) |
| `_microcompact` | Replace old, verbose tool results for compactable tools with `[result omitted]` to save context |
| `_apply_tool_result_budget` | Truncate any tool result that exceeds `max_tool_result_chars` |
| `_snip_history` | Drop old turns when the estimated context window is nearly full |

These are pre-call transformations on a copy — the persisted conversation in `messages` is never mutated.

---

## Loop Termination

The `AgentRunner.run()` loop ends when:

| Condition | `stop_reason` |
|---|---|
| LLM returns text with no tool calls | `"completed"` |
| `ask_user` tool is called | `"ask_user"` |
| LLM returns `finish_reason="error"` | `"error"` |
| `fail_on_tool_error=True` and a tool raises | `"tool_error"` |
| `max_iterations` reached | `"max_iterations"` |

---

## Key Files

| File | Role |
|---|---|
| [nanobot/agent/runner.py](../../nanobot/agent/runner.py) | Outer loop: LLM call → tool execution → repeat |
| [nanobot/providers/anthropic_provider.py](../../nanobot/providers/anthropic_provider.py) | Format conversion, response parsing |
| [nanobot/providers/base.py](../../nanobot/providers/base.py) | `LLMResponse`, `ToolCallRequest`, `should_execute_tools` |
| [nanobot/agent/tools/base.py](../../nanobot/agent/tools/base.py) | `Tool` ABC, `@tool_parameters` decorator, schema validation |
| [nanobot/agent/tools/registry.py](../../nanobot/agent/tools/registry.py) | Registration, definition generation, dispatch |

---

## Built-in Tools vs MCP Tools

### The terms, generally

**"Tool use"** (or "function calling") is the LLM-level protocol: the model emits a structured `tool_use` block containing a name and arguments; the host executes the corresponding function and feeds the result back. This is a capability of the LLM API itself — it says nothing about where the tool implementation lives or how it was wired in.

**MCP (Model Context Protocol)** is a separate, higher-level standard that defines how a *tool server* exposes capabilities to an *MCP client*. An MCP server can advertise tools, resources, and prompt templates over a transport (stdio, SSE, HTTP). The client connects, discovers what the server offers, and then calls those capabilities on demand. MCP is implementation-layer infrastructure; it has no bearing on the tool-use protocol the LLM sees.

The two concepts sit at different layers:

```
┌──────────────────────────────────────────────────┐
│                  LLM API layer                   │
│   "tool_use" / "function calling" protocol       │
│   Claude sees names + JSON Schema, returns calls │
└──────────────────┬───────────────────────────────┘
                   │ who provides the implementations?
        ┌──────────┴──────────┐
        ▼                     ▼
  Built-in tools          MCP tools
  (Python classes          (remote servers,
   in-process)             connected at startup)
```

Both ultimately appear to Claude as identical tool definitions. The distinction is invisible to the LLM.

---

### How nanobot specifically implements each

#### Built-in tools

Built-in tools are Python classes that subclass `Tool` and live inside the nanobot process. They are registered at agent startup and their `execute()` method runs **in-process** as an `async` coroutine. Examples: `read_file`, `exec`, `web_search`, `grep`.

Registration path:

```
AgentLoop.__init__()
  └── ToolRegistry.register(ReadFileTool())
      ToolRegistry.register(ExecTool())
      ...
```

The tool definition emitted to the LLM comes from `tool.to_schema()` → OpenAI `function` format → converted to Anthropic `input_schema` format by the provider.

#### MCP tools

MCP tools are capabilities exposed by an external **MCP server process** (or HTTP endpoint). At agent startup, `AgentLoop._connect_mcp()` ([loop.py:414](../../nanobot/agent/loop.py#L414)) calls `connect_mcp_servers()` which:

1. Opens a transport connection (stdio subprocess / SSE / streamable HTTP) for each configured server.
2. Calls `session.initialize()` and `session.list_tools()` over the MCP protocol.
3. Wraps each discovered tool in an `MCPToolWrapper` ([mcp.py:144](../../nanobot/agent/tools/mcp.py#L144)) and registers it in the same `ToolRegistry`.
4. Also discovers and registers `MCPResourceWrapper` (static URIs) and `MCPPromptWrapper` (template prompts).

The wrapper's `execute()` does a network call — `session.call_tool(original_name, arguments=kwargs)` — with a configurable timeout (default 30 s).

Name mangling: `mcp_{server_name}_{tool_name}`, sanitized to `[a-zA-Z0-9_-]`. This is how you can tell them apart in the tool list: MCP tools always start with `mcp_`.

Config schema ([config/schema.py:237](../../nanobot/config/schema.py#L237)):

```json
{
  "mcpServers": {
    "my_server": {
      "command": "npx",
      "args": ["-y", "@my/mcp-server"],
      "type": "stdio",
      "toolTimeout": 30,
      "enabledTools": ["*"]
    }
  }
}
```

`enabledTools` is a whitelist filter applied at registration time — tools not in the list are simply never registered and never appear in the LLM's tool list. `["*"]` (default) allows all tools from the server.

---

### Comparison: built-in vs MCP

#### Latency

| | Built-in | MCP |
|---|---|---|
| **Per-call overhead** | Negligible — in-process async call | Network RTT + JSON-RPC framing; typically 5–500 ms per call |
| **Startup cost** | Zero | One connection per server at agent startup; `session.initialize()` + `list_tools()` call |
| **Timeout** | Python exception propagates immediately | Configurable `tool_timeout` (default 30 s); `asyncio.wait_for` wraps each call |
| **Retry on transient error** | N/A (exception surfaces to runner) | One automatic retry for transient connection errors (`ClosedResourceError`, etc.) |

For tools that are called in a tight loop — iterating files, running grep — built-in is significantly faster because there is no process boundary or network involved. MCP overhead accumulates: ten calls to a remote MCP server add seconds to the turn.

#### Tool list length and relevance

Every registered tool — built-in or MCP — is sent to the LLM on every API call as part of the `tools=` parameter. This has two consequences:

**Token cost.** The tool definitions block is part of the prompt. A single tool definition with a detailed description and a complex schema can be 200–500 tokens. Fifty tools is 10,000–25,000 tokens sent on every request, before any conversation. `ToolRegistry.get_definitions()` caches the list between register/unregister calls, but the list is always sent in full.

**Decision quality.** Models perform better when the available tool list is tightly scoped to what is relevant for the task. A long tool list — especially one where many tools have similar-sounding names or overlapping descriptions — increases the chance the model picks the wrong one or hedges between them. MCP servers that expose dozens of tools all at once contribute disproportionately to this problem because a single server connection floods the registry.

**The MCP bulk problem.** A real-world MCP server commonly exposes 20–100 tools covering its entire API surface. Most of those are irrelevant to any single agent session — a GitHub MCP server might offer tools for issues, PRs, releases, actions, gists, orgs, and teams, but a coding agent only needs a handful of PR and file tools. When connected with `enabledTools: ["*"]`, *all* of those definitions land in the registry and are sent to the LLM on every turn. The cost compounds: token overhead from the definitions block, reduced cache hit rates, and a noisier decision space for the model — even for requests that have nothing to do with those tools.

The `enabledTools` config field is the primary mitigation: whitelist only the subset of an MCP server's tools that are actually needed. Leaving it at `["*"]` is the default but is rarely the right choice for large MCP servers.

#### Prompt caching

Tool definitions are a natural cache anchor because they are stable across turns of the same session. `AnthropicProvider._apply_cache_control()` places `cache_control: {"type": "ephemeral"}` markers at two positions in the tool list ([base.py:234](../../nanobot/providers/base.py#L234)):

1. The last **built-in** tool (the boundary between the sorted-stable built-in block and the MCP block).
2. The last tool overall (the tail of the MCP block).

This two-marker strategy means:
- If no MCP servers are connected: one marker at the tail of the built-in block.
- If MCP servers are connected: a marker at the built-in/MCP boundary allows the built-in definitions to be cached independently from the (potentially more volatile) MCP portion.

Built-in tools are sorted alphabetically and their schemas never change at runtime, so they cache perfectly across every request. MCP tools are also sorted, but their presence depends on which servers successfully connected, and `enabledTools` filter changes invalidate the cache. Any unregister/register call (e.g. an MCP server reconnect) clears `_cached_definitions` and forces a cold cache miss on the next LLM call.

#### Context window pressure

Tool results — not just definitions — consume context on every turn they remain in history. Built-in tools that return large outputs (file reads, grep matches) are subject to nanobot's context governance pipeline (`_microcompact`, `_apply_tool_result_budget`, `_snip_history`). MCP tool results go through exactly the same pipeline; there is no special treatment.

What differs is predictability: built-in tool output sizes are generally bounded and understood (you control the code). MCP tool output can be arbitrarily large and is outside your control — a remote tool might return megabytes of JSON. `max_tool_result_chars` (from `AgentRunSpec`) truncates any result beyond the configured limit before it is appended to history, but truncated results can confuse the model. Setting a conservative `toolTimeout` does not help with output size — the call can complete quickly but return a huge payload.

#### Operational risk

| | Built-in | MCP |
|---|---|---|
| **Process isolation** | None — a bug crashes the agent | Server runs in a separate process; crash is caught and returned as an error string |
| **Availability** | Always available if the process is running | Can fail to connect at startup or disconnect mid-session |
| **Schema drift** | Controlled by the codebase | Server can change its tool list between versions without your knowledge |
| **Security surface** | Narrower — you review the code | Wider — the server is a third-party dependency; SSRF and workspace guards apply but server-side behavior is opaque |

#### Summary table

| Dimension | Built-in | MCP |
|---|---|---|
| Execution location | In-process | Subprocess or remote HTTP |
| Per-call latency | ~0 ms | 5–500 ms + timeout risk |
| Schema defined by | Your Python class | Remote server (discovered at connect time) |
| Tool list control | Full — register exactly what you want | `enabledTools` filter; default is all tools |
| Cache friendliness | Excellent — stable, sorted, always the same | Good if tools are stable; any reconnect busts cache |
| Output size control | Predictable; you write the code | Unpredictable; `max_tool_result_chars` is the safety net |
| Deployment | Ships with nanobot | Requires a separate server process or network endpoint |
| Best for | Core agent capabilities, tight loops, low-latency ops | Third-party integrations, ecosystem tools, capabilities you don't want to re-implement |
