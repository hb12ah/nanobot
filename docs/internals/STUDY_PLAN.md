# nanobot Study Plan

## Big Picture

nanobot is an async Python AI agent framework. The core idea is simple: **messages come in from chat apps, get routed through a central bus to an agent loop, the loop calls an LLM and executes tools in a cycle, then responses flow back out through the bus to the channel.**

Everything is built around that one loop. The complexity you'll encounter is mostly at the edges (channel adapters, provider quirks, memory consolidation) — the core path stays small and readable by design.

```
User (Telegram / CLI / WebUI)
        ↓
   BaseChannel          ← platform adapter
        ↓ InboundMessage
   MessageBus           ← async queue (dead simple)
        ↓
   AgentLoop            ← session orchestrator
        |                 loads memory, skills, context
        ↓
   AgentRunner          ← LLM ↔ tool iteration
        ↓
   LLMProvider          ← Anthropic / OpenAI-compat / etc.
        ↑ tool results fed back in
   ToolRegistry         ← filesystem, shell, web, MCP...
        ↓ OutboundMessage
   MessageBus
        ↓
   BaseChannel          ← sends reply back

  Running in background:
   HeartbeatService     ← periodic proactive wake-up
   CronService          ← scheduled recurring tasks
   Dream                ← async memory consolidation
   AutoCompact          ← idle session compression
   SubagentManager      ← background task delegation
```

---

## Stage 1 — Data structures and config (30 min)

Read these first. They define the vocabulary everything else uses.

1. `nanobot/bus/events.py` — `InboundMessage` and `OutboundMessage`. Two dataclasses. Understand what travels through the system.
2. `nanobot/bus/queue.py` — `MessageBus`. ~45 lines. Two `asyncio.Queue`s. Trivial but central.
3. `nanobot/config/schema.py` — The Pydantic config tree. Skim it top to bottom so you know what knobs exist. Don't memorize — just recognize the shape.
4. `nanobot/providers/base.py` — `LLMResponse`, `ToolCallRequest`, `LLMProvider` abstract base. The interface every provider implements.

> **Confusing:** `config/schema.py` uses `to_camel` alias generators so every field has two names (camelCase for JSON, snake_case in Python). Don't be alarmed when you see `model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)` — it just means both `apiKey` and `api_key` work in config.

---

## Stage 2 — The core loop (1–2 hours)

This is the heart of the project.

5. `nanobot/agent/hook.py` — `AgentHook` and `AgentHookContext`. Read before runner/loop so you understand the callback surface.
6. `nanobot/agent/runner.py` — `AgentRunner`. The inner LLM ↔ tool loop. Read the `run()` method carefully. This is where tool calls happen, results get fed back, and the loop decides when to stop.
7. `nanobot/agent/loop.py` — `AgentLoop`. The session-level orchestrator that wraps runner. This is the largest file — read `process_message()` and `process_direct()` first, then trace what happens on each turn.
8. `nanobot/agent/context.py` — `ContextBuilder`. Assembles the system prompt from templates, memory, and skills before each turn.

> **Confusing:** `AgentLoop` and `AgentRunner` are easy to conflate. Mental model: `AgentLoop` is "per session/conversation" (loads memory, manages history, handles cron/heartbeat), while `AgentRunner` is "per LLM call" (runs the tool loop until the model stops calling tools). Runner is reused by Dream, subagents, and heartbeat too.

> **Confusing:** `_LoopHook` inside `loop.py` implements `AgentHook` to wire streaming/progress callbacks from the runner back out to channels. It's an internal implementation detail — don't confuse it with user-facing hook extension points.

---

## Stage 3 — Skills (30 min)

Skills belong here, right after you understand context assembly, because skills are injected as part of that context.

9. `nanobot/agent/skills.py` — `SkillsLoader`. Reads `SKILL.md` files from `nanobot/skills/` (built-in) and `~/.nanobot/workspace/skills/` (user-defined). Each skill has YAML frontmatter (description, requirements, `always` flag) and a markdown body.
10. Skim one built-in skill file, e.g. `nanobot/skills/memory/SKILL.md` or `nanobot/skills/cron/SKILL.md`, to see the format.
11. Glance at `nanobot/templates/` — `SOUL.md`, `TOOLS.md`, `AGENTS.md`, `HEARTBEAT.md`. These are Jinja2 markdown fragments rendered into the system prompt. Skills are injected alongside them.

> **Confusing:** Skills are not Python code — they are markdown documents the LLM reads. The agent decides which skill to invoke based on their descriptions; `SkillsLoader` just handles discovery and loading. Skills with `always: true` in frontmatter are always injected; others are listed as a summary and the agent fetches the full content via `read_file` only when needed.

> **Confusing:** "Skill" in nanobot means an agent prompt extension (a `SKILL.md` file). Don't confuse it with "tool" (a Python class with a `schema` and `execute()` method). Skills tell the agent *how* to use tools; tools give the agent *the ability* to act.

---

## Stage 4 — One provider and one channel (45 min)

Pick one of each to read in full. Don't read all of them.

**Provider (pick one):**
- `nanobot/providers/anthropic_provider.py` — most feature-complete, good to read
- `nanobot/providers/openai_compat_provider.py` — simpler, covers the majority of other providers

Then read `nanobot/providers/factory.py` to see how config selects the right one, and `nanobot/providers/registry.py` to see how providers are registered.

**Channel (pick one):**
- `nanobot/channels/websocket.py` — drives the WebUI; no external SDK dependency, easiest to follow
- `nanobot/channels/telegram.py` — representative of a real chat channel

Then read `nanobot/channels/manager.py` to see how all channels are started and how outbound messages are routed back to the right channel.

> **Confusing:** Channel classes have both async `run()` (the receive loop, blocking forever) and `send_message()` / `send_delta()` (for outbound). The two directions are independent — `run()` publishes to `bus.inbound`, and `ChannelManager` consumes from `bus.outbound` and calls `send_message()`. They don't directly call each other.

---

## Stage 5 — Session, memory, and autocompact (1 hour)

12. `nanobot/session/manager.py` — How conversation history is persisted to disk. Focus on `get_or_create()`, `save()`, and how the session key (`channel:chat_id`) works.
13. `nanobot/agent/memory.py` — Three things in one file:
    - `MemoryStore` — raw file I/O for `MEMORY.md` and `history.jsonl`
    - `Consolidator` — trims the in-session message list when it gets too long, writing older turns to disk
    - `Dream` — async background job that summarizes history into `MEMORY.md` using the LLM itself
14. `nanobot/agent/autocompact.py` — `AutoCompact`. When a session has been idle longer than `session_ttl_minutes`, it proactively summarizes the stale portion and replaces it with a compact summary. The last 8 messages are always kept verbatim. This prevents returning users from blowing up the context window.

> **Confusing:** "Memory" means two different things here. `Session.messages` = short-term conversation history (in RAM + session JSON on disk). `MEMORY.md` = long-term persistent facts on disk. `Consolidator` manages the former (trims it); `Dream` moves facts from the former to the latter. `AutoCompact` compresses idle sessions before a new turn starts.

> **Confusing:** `AutoCompact` and `Consolidator` both shrink the message history, but they trigger differently. `Consolidator` runs during an active turn when the context window gets too full. `AutoCompact` runs proactively on idle sessions before the user sends their next message, using a TTL clock.

---

## Stage 6 — Tools (30 min)

15. `nanobot/agent/tools/base.py` — `BaseTool` interface: `schema` dict + `async execute()`.
16. `nanobot/agent/tools/registry.py` — `ToolRegistry`: registration and lookup.
17. Skim `nanobot/agent/tools/shell.py` and `nanobot/agent/tools/filesystem.py` as representative examples.
18. `nanobot/agent/tools/spawn.py` — `SpawnTool`. This is how the main agent creates a subagent. Read it to understand the interface before reading the subagent manager.
19. `nanobot/agent/tools/mcp.py` — How MCP (Model Context Protocol) servers are discovered and wrapped as tools at runtime. Skim — detailed MCP protocol knowledge not required.

---

## Stage 7 — Subagents (45 min)

20. `nanobot/agent/subagent.py` — `SubagentManager`. Spawns separate `AgentRunner` instances for long-running background tasks. Each subagent gets its own tool registry, lifecycle hook (`_SubagentHook`), and status tracker (`SubagentStatus`). The main agent interacts with them via `SpawnTool` (create) and can poll or cancel via `SelfTool`.

> **Confusing:** Subagents are not separate processes — they are `asyncio` tasks running in the same event loop as the main agent. They share the same `LLMProvider` instance but have fully independent message histories and tool registries.

> **Confusing:** The subagent's `AgentRunner` uses `_SubagentHook` (logging/status only) — not `_LoopHook` — so it doesn't stream output back to the user channel directly. Results are returned to the spawning turn when the task completes.

---

## Stage 8 — Cron and heartbeat (30 min)

21. `nanobot/cron/service.py` — `CronService`. Schedules repeating and one-shot tasks. Jobs are persisted as JSON in `~/.nanobot/cron/`. The agent creates and manages them via `CronTool`.
22. `nanobot/heartbeat/service.py` — `HeartbeatService`. Periodically wakes the agent to check if there are outstanding tasks. Two-phase: Phase 1 makes a minimal LLM call with a virtual `heartbeat` tool to get a structured `skip`/`run` decision (avoids free-text parsing). Phase 2 only fires if Phase 1 returns `run`, running the task through the full agent loop. Reads `nanobot/templates/HEARTBEAT.md` for its instruction context.

> **Confusing:** Cron and heartbeat solve different problems. Cron runs *specific scheduled tasks* at configured times — it's user-controlled. Heartbeat runs *constantly in the background* and asks the LLM "do you have anything to do right now?" — it's the agent's autonomy mechanism. A cron job can trigger via heartbeat, but heartbeat itself has no schedule beyond its polling interval.

---

## Stage 9 — Python SDK and putting it all together (20 min)

23. `nanobot/nanobot.py` — `Nanobot` facade. ~130 lines. `Nanobot.from_config()` wires the entire system (provider, bus, loop, all config) and exposes a single `await bot.run("…")` method. The best capstone read — it shows how every piece you've studied fits together.

---

## What to ignore on a first run

- **`bridge/`** — TypeScript WhatsApp sidecar. Only relevant if working on WhatsApp.
- **`webui/`** — React frontend. Skip unless touching the UI. The backend it talks to is in `nanobot/channels/websocket.py` and `nanobot/api/server.py`.
- **`nanobot/channels/`** (most of them) — Read one or two for understanding; the rest (dingtalk, feishu, qq, wecom, matrix, etc.) are the same pattern repeated per platform.
- **`nanobot/providers/`** (most of them) — Same: read one or two, skip the rest until you need a specific backend.
- **`tests/`** — Useful reference when working on a specific area, but don't attempt a top-to-bottom read.

---

## Suggested reading order (condensed)

```
events.py → queue.py → config/schema.py → providers/base.py
→ agent/hook.py → agent/runner.py → agent/loop.py → agent/context.py
→ agent/skills.py → one SKILL.md → templates/ (skim)
→ one provider → providers/factory.py + registry.py
→ one channel → channels/manager.py
→ session/manager.py → agent/memory.py → agent/autocompact.py
→ tools/base.py + registry.py → tools/shell.py → tools/spawn.py
→ agent/subagent.py
→ cron/service.py → heartbeat/service.py
→ nanobot/nanobot.py   ← ties it all together
```

---

## Glossary

**AgentHook** — Lifecycle callback interface. Implement `before_iteration`, `on_stream`, `after_iteration`, etc. to observe or modify agent runs without touching the core loop.

**AgentHookContext** — Mutable state snapshot passed to every hook method per iteration: current messages, LLM response, tool calls, tool results, token usage.

**AgentLoop** — The per-session orchestrator in `agent/loop.py`. Owns session history, memory, skills, cron, heartbeat, and subagents for one conversation. Calls `AgentRunner` once per turn.

**AgentRunner** — The inner LLM ↔ tool iteration engine in `agent/runner.py`. Calls the LLM, executes tool calls, feeds results back, and repeats until the model stops calling tools or max iterations is hit. Shared by the main loop, Dream, subagents, and heartbeat.

**AutoCompact** — Proactively summarizes idle sessions before a new turn begins. Triggered by `session_ttl_minutes` elapsed since last activity. Keeps the last 8 messages verbatim; compresses the rest.

**BaseChannel** — Abstract base class in `channels/base.py`. Every chat platform (Telegram, Discord, WebSocket, etc.) subclasses this, bridging the platform SDK to `MessageBus`.

**BaseTool** — Abstract tool interface: a `schema` (JSON Schema dict) and `async execute()`. Every tool the agent can call implements this.

**ChannelManager** — Starts all enabled channels, fans outbound messages to the right one based on the channel name in `OutboundMessage`.

**Consolidator** — In-session message trimmer. During an active turn, when the running context exceeds `context_window_tokens`, it writes older messages to `history.jsonl` and removes them from the live list.

**ContextBuilder** — Assembles the full system prompt before each turn: identity templates, memory, tools documentation, skills, and workspace info.

**CronJob / CronService** — User-controlled scheduled tasks, persisted as JSON in `~/.nanobot/cron/`. The agent creates and manages them via `CronTool`.

**Dream** — Async background LLM job that reads recent `history.jsonl` entries and rewrites `MEMORY.md` to consolidate important facts. Runs on a configurable interval (default every 2 hours).

**HeartbeatService** — Periodic background service that wakes the agent to check for outstanding tasks. Phase 1 makes a cheap LLM call with a virtual `heartbeat` tool to get a structured `skip`/`run` decision. Phase 2 only runs the full agent loop when Phase 1 returns `run`.

**InboundMessage / OutboundMessage** — The two event types that travel through `MessageBus`. `InboundMessage` carries a user message from a channel to the agent; `OutboundMessage` carries the agent's reply back.

**Iteration** — One LLM call within a single turn. A turn with multiple tool calls has multiple iterations: each call → tool result → next call is one iteration.

**LLMProvider** — Abstract interface in `providers/base.py`. Every backend (Anthropic, OpenAI-compat, Azure, Bedrock, etc.) implements this. `factory.py` selects the right one from config.

**MCP (Model Context Protocol)** — Standard for attaching external tool servers to an LLM. nanobot loads MCP servers defined in config and wraps each exposed tool as a `BaseTool` at runtime via `tools/mcp.py`.

**MEMORY.md** — The persistent long-term memory file on disk in the workspace. Written by `Dream`, read by `MemoryStore` and injected into context by `ContextBuilder`.

**MemoryStore** — Pure file I/O layer for `MEMORY.md`, `history.jsonl`, `SOUL.md`, and `USER.md`. No logic — just reads and writes.

**MessageBus** — Two `asyncio.Queue`s (inbound + outbound) that decouple channels from the agent. Channels push to inbound; the agent consumes inbound and pushes to outbound; `ChannelManager` consumes outbound.

**Session** — One conversation's in-memory message list plus metadata, keyed as `channel:chat_id`. Persisted atomically to `~/.nanobot/sessions/` by `SessionManager`.

**Session key** — String in the form `channel:chat_id` (e.g. `telegram:123456789`). Uniquely identifies one conversation. The special key `unified:default` merges all channels into one shared history when `unified_session` is enabled.

**SKILL.md** — Markdown file with YAML frontmatter that teaches the agent how to use a specific capability (e.g. memory, cron, GitHub). Not Python code — injected as text into the system prompt.

**SkillsLoader** — Discovers and loads `SKILL.md` files from `nanobot/skills/` (built-in) and `~/.nanobot/workspace/skills/` (user). Skills with `always: true` are always injected; others are listed as a summary and fetched on demand.

**SOUL.md** — Jinja2 template defining the agent's identity, persona, and core instructions. Rendered once at loop startup.

**SpawnTool** — The tool the agent calls to create a subagent. Passes a task description and receives a task ID; the subagent runs asynchronously.

**SubagentManager** — Manages background `asyncio` tasks, each running an independent `AgentRunner`. Subagents are not separate processes; they share the event loop and LLM provider but have isolated histories and tool registries.

**ToolRegistry** — Central registry of all tools available in a session. `AgentLoop` registers built-in tools at startup; MCP tools are registered dynamically.

**Turn** — One complete user message → agent response cycle. May involve multiple iterations (LLM calls) if the model makes tool calls before producing a final response.

**Unified session** — When `unified_session: true` in config, all channels share one conversation history under the key `unified:default` instead of one session per channel.
