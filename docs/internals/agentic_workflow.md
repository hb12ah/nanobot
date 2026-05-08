# Agentic Workflow — How nanobot Thinks and Acts

Yes — at its core, nanobot is a **ReAct loop** (Reason + Act). But it is
wrapped in several layers that handle concurrency, memory, recovery, and
delegation. This document traces a message from arrival to reply, explaining
each layer along the way.

---

## 1. The Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│  Channel (Telegram / Discord / CLI / WebSocket / Cron …)    │
└───────────────────────────┬─────────────────────────────────┘
                            │  InboundMessage
                            ▼
                    ┌───────────────┐
                    │  MessageBus   │  (async queue)
                    └──────┬────────┘
                           │
                           ▼
                    ┌───────────────┐
                    │  AgentLoop    │  outer dispatcher
                    │  (loop.py)    │  ← manages sessions,
                    └──────┬────────┘    commands, concurrency
                           │
                           ▼
                    ┌───────────────┐
                    │  AgentRunner  │  inner ReAct loop
                    │  (runner.py)  │  ← LLM ↔ tools
                    └──────┬────────┘
                           │  OutboundMessage
                           ▼
                    ┌───────────────┐
                    │  MessageBus   │
                    └──────┬────────┘
                           │
                           ▼
                    ┌───────────────┐
                    │  Channel      │  sends reply to user
                    └───────────────┘
```

The **MessageBus** is the only communication surface between channels and the
agent. Channels never call the agent directly; the agent never knows which
channel it is talking to unless it reads `msg.channel`.

---

## 2. AgentLoop — The Outer Dispatcher

`AgentLoop.run()` is a single `while True` that reads from the inbound queue
and dispatches each message as an `asyncio.Task`. This keeps the loop
responsive to priority commands like `/stop` even while a long-running turn
is in progress.

```
AgentLoop.run()
    │
    ├─ wait for InboundMessage from bus
    │
    ├─ [priority command?] ──► dispatch inline, continue
    │
    ├─ [session already active?]
    │       └─ route into pending_queue (mid-turn injection)
    │
    └─ create asyncio.Task(_dispatch(msg))
              │
              ▼
         _dispatch(msg)
              │
              ├─ acquire session lock  (one turn at a time per session)
              ├─ acquire concurrency gate  (default: max 3 concurrent sessions)
              ├─ register pending_queue  (captures follow-up messages mid-turn)
              │
              └─► _process_message(msg)
                        │
                        ├─ refresh provider config (hot-swap model without restart)
                        ├─ restore crash checkpoint if present
                        ├─ run slash-command router
                        ├─ run auto-compact / consolidation if needed
                        ├─ build initial_messages (system prompt + history + user turn)
                        ├─ persist user message early (crash-safe write-ahead)
                        │
                        └─► _run_agent_loop(initial_messages)   ←── enters AgentRunner
```

### Session isolation

Each conversation is keyed as `channel:chat_id` (e.g. `telegram:1234567`).
An `asyncio.Lock` per session ensures turns are processed serially — a second
message from the same user always waits for the current turn to finish rather
than racing against it. Messages that arrive while a turn is running are
placed in a `pending_queue` and injected mid-turn as described in §4.

---

## 3. AgentRunner — The Inner ReAct Loop

This is the classic ReAct pattern: **call the LLM → execute tools → repeat**.

```
AgentRunner.run(spec)
│
└─ for iteration in range(max_iterations):
      │
      ├─ 1. CONTEXT GOVERNANCE
      │       microcompact old tool results
      │       apply token budget (snip oldest messages if over limit)
      │       drop/backfill orphaned tool call ↔ result pairs
      │
      ├─ 2. LLM CALL  →  response
      │       (streaming or polling, with retry logic)
      │
      ├─ 3a. response has tool_calls?
      │        │
      │        ├─ append assistant message (with tool_calls)
      │        ├─ emit crash checkpoint  ← durable state written here
      │        ├─ execute tools (concurrent when safe, serial for ask_user)
      │        ├─ append tool result messages
      │        ├─ drain pending injections (mid-turn user follow-ups)
      │        └─ continue → next iteration
      │
      └─ 3b. no tool_calls  →  final text response
               │
               ├─ drain pending injections (if any, continue loop)
               ├─ emit crash checkpoint
               └─ break → return AgentRunResult
```

### Stop reasons

| `stop_reason`         | Meaning                                              |
|-----------------------|------------------------------------------------------|
| `completed`           | Normal: LLM returned a final text response           |
| `ask_user`            | Agent called `ask_user` tool — waiting for input     |
| `max_iterations`      | Hit the turn limit without a final text response     |
| `tool_error`          | A tool raised a fatal exception                      |
| `error`               | LLM returned `finish_reason=error`                   |
| `empty_final_response`| LLM returned blank text after retries                |

---

## 4. Mid-Turn Message Injection

A distinctive nanobot feature: **messages can arrive while a turn is running**
and be injected into the ongoing conversation rather than queued as a separate
turn. This matters for subagent results — a spawned background task completes
while the parent is still reasoning, and its output is folded in seamlessly.

```
User message A arrives
    │
    ▼
Turn A starts (pending_queue registered)

    ... LLM calls spawn_tool ...

    Subagent finishes → publishes InboundMessage(channel="system")
        │
        └─► routed to pending_queue instead of spawning a new task

    ... after next tool execution ...

    AgentRunner calls injection_callback()
        └─► drains pending_queue → appends as user message → continues loop

Turn A ends (pending_queue unregistered)
```

Injections are capped at 3 messages per call and 5 injection cycles per turn
to prevent infinite loops.

---

## 5. Context Assembly

Before every turn, `ContextBuilder` assembles what the LLM sees:

```
System prompt
├─ Identity / SOUL.md  (who the agent is)
├─ Bootstrap files     (AGENTS.md, USER.md, TOOLS.md)
├─ Memory              (MEMORY.md — persistent facts)
├─ Always-on skills    (markdown skill fragments loaded unconditionally)
├─ Skills summary      (list of available on-demand skills)
└─ Recent history      (raw history.jsonl entries since last Dream run)

Conversation messages
├─ Session history     (up to max_messages, token-budgeted)
│   └─ Consolidator summary prefix  (if history was compacted)
└─ Current user message
    ├─ Runtime context block  (timestamp, channel, chat_id — stripped before save)
    └─ User text / media
```

The runtime context block is prepended to every user turn but stripped before
the session is saved, so it never accumulates in history.

---

## 6. Memory — Three Layers

nanobot has three distinct memory mechanisms that work at different time scales:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer        │  Where          │  Lifetime     │  Writer   │
├───────────────┼─────────────────┼───────────────┼───────────┤
│ Session       │ ~/.nanobot/     │ Per session   │ AgentLoop │
│ history       │ sessions/*.jsonl│ (survives     │ _save_turn│
│               │                 │  restarts)    │           │
├───────────────┼─────────────────┼───────────────┼───────────┤
│ Consolidator  │ In-memory +     │ Per session,  │ Consolidat│
│ summary       │ session prefix  │ compacts when │ or        │
│               │                 │ over token    │           │
│               │                 │ threshold     │           │
├───────────────┼─────────────────┼───────────────┼───────────┤
│ Dream /       │ memory/         │ Permanent     │ Dream     │
│ MEMORY.md     │ MEMORY.md       │ across all    │ (async,   │
│               │                 │ sessions      │ every 2h) │
└───────────────┴─────────────────┴───────────────┴───────────┘
```

**Consolidator** compacts a single session's history when it grows too large
for the context window. It runs synchronously before each turn if needed, and
asynchronously in the background after each turn.

**Dream** runs as a cron job (default every 2 hours). It reads raw
`history.jsonl` entries that haven't yet been processed, summarizes them via
an LLM call, and appends the summary to `MEMORY.md`. This is how short-term
observations become long-term memory.

---

## 7. Subagents — Delegated Background Tasks

When the agent calls the `spawn` tool, `SubagentManager` creates a fresh
`AgentRunner` with its own tool registry and runs it in a background
`asyncio.Task`. When done, it announces back via the bus as a `system`
channel message, which is injected mid-turn (§4) or processed as a new turn.

```
Parent AgentRunner
    │
    └─ spawn_tool(task="…", label="…")
            │
            ▼
       SubagentManager.spawn()
            │
            ├─ create isolated ToolRegistry
            ├─ run AgentRunner.run(spec) as background task
            │
            └─ on completion:
                 bus.publish_inbound(InboundMessage(
                     channel="system",
                     sender_id="subagent",
                     chat_id="<channel>:<chat_id>",
                     content=<result>,
                 ))
```

Subagents do **not** have access to session history, memory write tools, or
the `spawn` tool itself (no recursive spawning). They are isolated compute
workers, not nested agents.

---

## 8. Crash Recovery

nanobot writes a **checkpoint** before executing each batch of tool calls.
If the process dies mid-turn, the next message to that session triggers
`_restore_runtime_checkpoint`, which replays the assistant message and
completed tool results from the checkpoint into session history, and marks
any interrupted tool calls as errored. The turn is then closed cleanly before
processing the new message.

```
Before tools execute:
    checkpoint = {assistant_message, completed_tool_results=[], pending_tool_calls=[...]}
    → written to session.metadata

After tools execute:
    checkpoint = {assistant_message, completed_tool_results=[...], pending_tool_calls=[]}
    → updated in session.metadata

On final response:
    checkpoint cleared
```

---

## 9. Hook System

`AgentHook` is a lifecycle interface injected into `AgentRunner`. It decouples
product concerns (streaming, progress indicators, logging) from the execution
core. The main hooks:

| Hook method           | When called                              |
|-----------------------|------------------------------------------|
| `before_iteration`    | Start of each ReAct iteration            |
| `on_stream` / `on_stream_end` | During / after LLM token streaming |
| `before_execute_tools`| Just before tool calls execute           |
| `after_iteration`     | End of each iteration                    |
| `finalize_content`    | Post-processing final LLM text           |

`_LoopHook` (used by `AgentLoop`) drives streaming to channels, emits
progress hints, and logs token usage. `_SubagentHook` logs tool calls for
background tasks. The Python SDK (`SDKCaptureHook`) captures the full
conversation for programmatic use.

---

## 10. Is It "Just ReAct"?

The `AgentRunner.run()` for-loop is a textbook ReAct loop. What makes nanobot
more than that:

| Feature                     | Where                                    |
|-----------------------------|------------------------------------------|
| Multi-session concurrency   | `AgentLoop` + per-session `asyncio.Lock` |
| Mid-turn message injection  | `pending_queue` + `injection_callback`   |
| Crash recovery              | Checkpoint in `session.metadata`         |
| Rolling context management  | `Consolidator` + `_snip_history`         |
| Long-term memory            | `Dream` + `MEMORY.md`                    |
| Delegated background work   | `SubagentManager` + `spawn` tool         |
| Scheduled autonomous tasks  | `CronService` → injects system messages  |
| Hot model/provider swap     | `_apply_provider_snapshot` per-turn      |
| Pluggable channel adapters  | `BaseChannel` + `MessageBus`             |
