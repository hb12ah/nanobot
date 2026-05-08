# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Install (development)**
```bash
pip install -e ".[dev]"
```

**Run tests**
```bash
pytest                              # all tests
pytest tests/agent/test_loop.py    # single file
pytest -k "test_consolidator"      # by name pattern
pytest --cov=nanobot                # with coverage
```

**Lint / format**
```bash
ruff check nanobot/
ruff format nanobot/
```

**Run the agent (CLI)**
```bash
nanobot onboard      # first-time setup
nanobot agent        # interactive chat
nanobot gateway      # start WebSocket/HTTP gateway for channels and WebUI
```

**WebUI (dev)**
```bash
# terminal 1
nanobot gateway

# terminal 2
cd webui
bun install
bun run dev         # opens at http://127.0.0.1:5173

# build for packaging (writes to nanobot/web/dist/)
bun run build

# run webui tests
bun run test
```

## Architecture

nanobot is an async Python AI agent. Everything flows through a central message bus:

```
Channel (Telegram, Discord, etc.)
    ↓ InboundMessage
  MessageBus (bus/queue.py)
    ↓
  AgentLoop (agent/loop.py)      ← orchestrates per-turn processing
    ↓
  AgentRunner (agent/runner.py)  ← tool-call / LLM iteration loop
    ↓
  LLMProvider (providers/)       ← Anthropic, OpenAI-compat, Azure, etc.
    ↓ OutboundMessage
  MessageBus
    ↓
  ChannelManager (channels/manager.py) → back to the channel
```

### Key modules

**`nanobot/agent/loop.py` — `AgentLoop`**  
The main per-session controller. Reads from `MessageBus.inbound`, assembles context (system prompt, memory, tools), delegates to `AgentRunner`, then writes to `MessageBus.outbound`. Also drives memory consolidation, cron, and heartbeat.

**`nanobot/agent/runner.py` — `AgentRunner`**  
The inner tool-execution loop. Calls the LLM, executes tool calls, feeds results back, repeats until `stop` or max iterations. Shared by the main loop, Dream, and subagents.

**`nanobot/agent/hook.py` — `AgentHook`**  
Lifecycle callbacks (`before_iteration`, `on_stream`, `after_iteration`, etc.) injected into `AgentRunner`. The main loop uses `_LoopHook`; the Python SDK uses `SDKCaptureHook`; channel streaming uses per-channel hooks.

**`nanobot/providers/`**  
LLM backends. All implement `LLMProvider` from `base.py`. `factory.py` selects the right one from config. Adding a provider means subclassing `LLMProvider` and registering it in `registry.py`.

**`nanobot/channels/`**  
Chat platform adapters. Each subclasses `BaseChannel` (channels/base.py), bridges the platform SDK to `MessageBus`, and handles send/receive. Channels are auto-discovered by name (pkgutil) or via `nanobot.channels` entry points for external plugins.

**`nanobot/session/manager.py` — `SessionManager` / `Session`**  
Persists per-conversation message history to `~/.nanobot/sessions/`. Sessions are keyed as `channel:chat_id`. Atomic writes with `filelock`.

**`nanobot/agent/memory.py` — `MemoryStore`, `Consolidator`, `Dream`**  
Three-layer memory: raw `MEMORY.md` file I/O (`MemoryStore`), in-session rolling consolidation (`Consolidator`), and periodic async background summarization into `MEMORY.md` (`Dream`). Dream runs as a cron job on a configurable interval (default every 2 h).

**`nanobot/cron/service.py` — `CronService`**  
Schedules repeating and one-shot tasks. Jobs are persisted as JSON in `~/.nanobot/cron/`. The agent can create/delete/list cron jobs via the `CronTool`.

**`nanobot/config/schema.py`**  
Pydantic models for `~/.nanobot/config.json`. All keys accept both camelCase and snake_case. The root `Config` object is the single source of truth for runtime settings.

**`nanobot/nanobot.py` — `Nanobot`**  
High-level Python SDK facade. `Nanobot.from_config()` → `await bot.run("…")`. Used by the Python SDK and subagents.

**`nanobot/agent/tools/`**  
Built-in tools registered in `ToolRegistry`. Each tool is a class with a `schema` (JSON Schema) and an `async def execute(...)` method. MCP tools are loaded dynamically and injected at runtime via `tools/mcp.py`.

**`nanobot/templates/`**  
Jinja2 markdown templates for system-prompt sections: `SOUL.md` (identity), `TOOLS.md` (tool docs), `AGENTS.md` (subagent docs), `HEARTBEAT.md`, `USER.md`. Rendered at loop startup and injected as context.

**`nanobot/skills/`**  
Markdown-defined skill files (`SKILL.md`). The agent loads them as context fragments on demand. Built-in skills live here; users can add custom skills to `~/.nanobot/skills/`.

**`bridge/`**  
TypeScript bridge for the WhatsApp channel (uses `@whiskeysockets/baileys`). Runs as a sidecar process communicating over `stdin`/`stdout` with the Python WhatsApp channel.

**`webui/`**  
React 18 / TypeScript / Vite frontend. Communicates with the gateway via the WebSocket multiplex protocol. Build output goes to `nanobot/web/dist/` and is bundled into the wheel.

## Branching

| Branch | Purpose |
|--------|---------|
| `main` | Stable — bug fixes and docs only |
| `nightly` | Experimental — new features, refactors |

Target `nightly` for new features; target `main` for bug fixes. Stable features are cherry-picked from `nightly` to `main` roughly weekly.

## Code conventions

- Python ≥ 3.11, `asyncio` throughout, `pytest` with `asyncio_mode = "auto"`
- Line length 100 (`ruff`), rules E, F, I, N, W (E501 ignored)
- Config models use `pydantic` v2 with `to_camel` alias generator — both camelCase and snake_case keys are accepted in JSON
- Logging via `loguru`; bind the channel name with `logger.bind(channel=self.name)` in channel classes
- Optional heavy dependencies (discord, matrix, wecom, msteams, pdf) are in `[project.optional-dependencies]` — import them lazily inside the relevant module
