# AgentLoop deep-dive (`nanobot/agent/loop.py`)

## TL;DR

`AgentLoop` is the **outer product shell** that wraps `AgentRunner`. It owns everything the runner deliberately knows nothing about: sessions, channels, users, memory, tools registration, concurrency, and crash recovery. One loop instance runs for the lifetime of the process and handles all sessions concurrently.

The call stack for a normal turn is:

```
run()                   ← polls the bus; spawns one asyncio task per message
  _dispatch()           ← owns the pending queue and concurrency gate
    _process_message()  ← session setup, prompt assembly, post-turn persistence
      _run_agent_loop() ← builds AgentRunSpec, calls runner.run(), returns result
        AgentRunner.run() ← the actual LLM/tool loop (see runner_explained.md)
```

`run()` is non-blocking — it spawns `_dispatch` as an asyncio task and immediately loops back to consume the next message. This means multiple turns can run concurrently across different sessions:

```
run() loops forever
  │
  ├── pulls msg A → spawns Task-A → loops immediately
  ├── pulls msg B (same session, Task-A still running) → pending queue → loops
  ├── pulls msg C (different session) → spawns Task-C → loops
  │
  Task-A                              Task-C
    _dispatch()                         _dispatch()
      _process_message()                  _process_message()
        _run_agent_loop()                   _run_agent_loop()
          runner.run()                        runner.run()
            iteration 1                         ...
            [drains msg B from queue]
            iteration 2
            final answer
      publish outbound
    finally: queue empty, nothing republished
```

`_dispatch` and `_process_message` each handle exactly one triggering message — the one that started the turn. `runner.run()` can handle more: it checks the pending queue at natural pause points (after tool rounds, after the final answer) and folds in any follow-up messages that have arrived by then, continuing the turn rather than ending it. The turn boundary is not "one user message → one response" but "keep going until there is nothing left to respond to."

**Subagents** are not part of the main agent's session. Each subagent is a stateless, fire-and-forget `AgentRunner.run()` call with its own fresh message list and tool registry — no `SessionManager`, no history, no persistence. When a subagent finishes, it publishes a `channel="system"` inbound message to the bus. The main loop picks this up like any other message and routes it through `_process_message`'s system path, which persists the result into the session and lets the main agent respond to it.

The main turn stays alive to receive the subagent result because `_drain_pending` (the injection callback in `_run_agent_loop`) has a special case: if the pending queue is empty but subagents are still running for this session, it blocks up to 300 s waiting for one to finish:

```
main turn running runner.run()
  → LLM calls spawn tool → SubagentManager.spawn() → asyncio.Task created
  → runner loop continues, LLM writes "I've started the task" answer
  → _drain_pending: queue empty, but subagent still running → BLOCKS (up to 300 s)
  → subagent finishes → publishes channel="system" InboundMessage
  → InboundMessage routed to pending queue (mid-turn injection)
  → _drain_pending unblocks, returns the announcement
  → runner loop continues with subagent result injected into context
  → LLM reads result, writes final response
```

---

## `_LoopHook` (lines 68–174)

`_LoopHook` is the concrete `AgentHook` injected into `AgentRunner` for each turn. It bridges runner lifecycle events back to the channel (streaming, progress) and to the loop (logging, tool context).

**Constructor (lines 71–94):** stores references to the parent loop, the streaming callbacks (`on_stream`, `on_stream_end`), the progress callback (`on_progress`), channel routing info, and an empty `_stream_buf` for incremental `<think>` stripping.

**`wants_streaming` (line 96):** returns `True` if `on_stream` was provided. The runner uses this to avoid calling streaming-specific paths when no one is listening.

**`on_stream` (lines 99–107):** strips `<think>` blocks *incrementally* from a running buffer. Only the new clean text beyond what was already forwarded is sent to the caller. This ensures thinking tokens from models like DeepSeek-R1 never reach the user.

**`on_stream_end` (lines 109–112):** calls `_on_stream_end(resuming=...)`, then resets `_stream_buf` for the next segment.

**`before_iteration` (lines 114–120):** stores the current iteration number on the loop; logs it with the session key.

**`before_execute_tools` (lines 122–147):** fires just before tools run. Emits tool-start progress events to the channel (formatted tool names/arguments). Logs each tool call. Calls `_set_tool_context` to point routing-aware tools at the current channel/chat.

**`after_iteration` (lines 149–170):** emits tool-finish progress events if the channel supports them. Logs LLM token usage.

**`finalize_content` (line 172–173):** strips `<think>` blocks from the final response text before it's used.

---

## `AgentLoop.__init__` (lines 191–324)

The constructor wires up every subsystem and stores them as attributes.

| Attribute | What it is |
|-----------|-----------|
| `bus` | The `MessageBus` — source of inbound messages, sink for outbound |
| `provider` / `model` | Active LLM backend and model name (can change at runtime) |
| `workspace` | Root path; all file tools are anchored here |
| `sessions` | `SessionManager` — reads/writes per-session JSONL history |
| `tools` | `ToolRegistry` — shared across all sessions |
| `runner` | `AgentRunner(provider)` — the inner engine |
| `subagents` | `SubagentManager` — runs background agents |
| `consolidator` / `auto_compact` | History token management: rolling trim and TTL-based compaction |
| `dream` | `Dream` — background async memory summarizer |
| `context` | `ContextBuilder` — assembles system prompts, history, and user content |
| `commands` | `CommandRouter` — handles slash commands before the LLM sees them |
| `_pending_queues` | `dict[session_key, Queue]` — mid-turn injection queues, one per active session |
| `_active_tasks` | `dict[session_key, list[Task]]` — tracks running tasks for `/stop` |
| `_concurrency_gate` | Optional `Semaphore` — limits total concurrent requests (default 3, env `NANOBOT_MAX_CONCURRENT_REQUESTS`) |
| `_file_state_store` | `FileStateStore` — per-session file read/write tracking for tools |
| `_extra_hooks` | Additional `AgentHook` instances injected by the caller (e.g. SDK hooks) |

Lines 318–324: after construction, `_register_default_tools()` populates the tool registry and `CommandRouter` is set up with built-in slash commands.

---

## Setup helpers (lines 326–434)

**`_sync_subagent_runtime_limits` (line 326):** copies `self.max_iterations` to `subagents.max_iterations`. Called at the start of each `_run_agent_loop` so runtime config changes propagate before the next turn.

**`_apply_provider_snapshot` (lines 330–346):** swaps the active provider and model on the loop, runner, subagents, consolidator, and dream all at once. No-ops if nothing changed.

**`_refresh_provider_snapshot` (lines 348–358):** called at the start of every `_process_message`. Loads the current config snapshot from disk; calls `_apply_provider_snapshot` only if the signature changed. This is how live config edits take effect without restarting.

**`_register_default_tools` (lines 360–412):** populates the shared `ToolRegistry`. Always registers core file, search, message, and spawn tools. Conditionally adds `ExecTool`, `WebSearchTool`/`WebFetchTool`, `CronTool`, and `MyTool` based on config. When `restrict_to_workspace` or sandbox is on, file tools get `allowed_dir=workspace` and reject paths outside it.

**`_connect_mcp` (lines 414–434):** connects to configured MCP servers and registers their tools into the shared registry. Lazy (first message) and one-shot. Failures are logged as warnings and retried on the next message.

---

## Small per-turn utilities (lines 436–530)

These are short helpers called from `_dispatch`, `_process_message`, or `_LoopHook`. None deserve a section of their own.

**`_set_tool_context` (lines 436–464):** updates routing-aware tools (message, spawn, cron, my) with the current channel, chat ID, and session key so they know where to send output.

**`_strip_think` (lines 466–473):** static helper — removes `<think>…</think>` blocks from a string; returns `None` if the result is empty.

**`_runtime_chat_id` (lines 475–478):** static helper — returns `metadata["context_chat_id"]` if present, otherwise `msg.chat_id`. Used to give the model a stable chat ID even when routing is overridden.

**`_tool_hint` (lines 480–484):** formats tool calls into a concise human-readable string for progress cards (e.g. `"read_file, exec_command"`), truncated to `tool_hint_max_length`.

**`_dispatch_command_inline` (lines 486–499):** dispatches a slash command without creating a task — builds a `CommandContext`, calls the dispatch function, publishes the result. Used for priority and non-priority commands that arrive while a session task is already running.

**`_cancel_active_tasks` (lines 501–512):** cancels all active tasks and subagents for a session key. Used by `/stop`.

**`_effective_session_key` (lines 514–518):** returns the session key for routing. In unified-session mode, all messages share `"unified:default"` regardless of chat ID.

**`_replay_token_budget` (lines 520–530):** computes how many tokens of history can be loaded into the prompt — context window minus reserved output tokens and a 1024-token safety margin.

---

## `_run_agent_loop` (lines 532–674)

Builds the `AgentRunSpec` and calls `runner.run()`. This is the seam between `AgentLoop` and `AgentRunner`.

**Lines 557–572 — hook construction:** `_LoopHook` is built for this turn. If `_extra_hooks` exist, combined with `CompositeHook`.

**Lines 574–577 — checkpoint closure:** `_checkpoint` saves in-flight turn state to `session.metadata` after each tool round. Passed to the runner as `checkpoint_callback`.

**Lines 579–637 — `_drain_pending` closure:** the `injection_callback` for the runner. Normally drains whatever is immediately available in the pending queue. Special case: if the queue is empty but subagents are still running for this session, blocks up to 300 s for a result — keeping the runner loop alive so subagent completions are processed in-order.

**Lines 642–661 — `AgentRunSpec` construction:** all limits, callbacks, and flags assembled and handed to `runner.run()`.

**Lines 664–674 — post-run handling:** updates `_last_usage`. For `max_iterations`, pushes final content through stream callbacks so streaming channels don't leave an empty card. Returns `(final_content, tools_used, messages, stop_reason, had_injections)`.

---

## `run()` — the main event loop (lines 676–749)

Runs forever (`while self._running`), polling the bus and dispatching messages as tasks.

**Lines 684–699 — bus poll:** awaits `bus.consume_inbound()` with a 1 s timeout. On timeout, `auto_compact.check_expired()` runs session cleanup and the loop continues. Real `CancelledError` is re-raised; other exceptions are swallowed with a warning.

**Lines 701–707 — priority commands:** matched commands (e.g. `/stop`) are dispatched inline immediately — no task created, so `/stop` can cancel a running task without queuing behind it.

**Lines 708–739 — session routing:** if a pending queue already exists for this session key (a task is active), the new message is routed there for mid-turn injection. Non-priority slash commands are dispatched inline even if a task is running. If the queue is full, falls back to spawning a new task so the message isn't dropped.

**Lines 742–749 — task creation:** a new `asyncio.Task` wrapping `_dispatch(msg)` is created and tracked in `_active_tasks[key]`. A done-callback removes it when finished.

---

## `_dispatch()` (lines 751–887)

Called inside each task spawned by `run()`. Owns the session lock, concurrency gate, and pending queue lifecycle.

**Lines 756–762 — lock, gate, and pending queue:** acquires the per-session lock (one task per session at a time) and the global semaphore (max concurrent requests). Registers a fresh `Queue(maxsize=20)` in `_pending_queues` for this session so follow-up messages have somewhere to land.

**Lines 768–797 — streaming setup:** if `msg.metadata["_wants_stream"]` is true (WebSocket channel), builds `on_stream` and `on_stream_end` closures. A `stream_segment` counter increments on each `on_stream_end` call, giving each tool-round segment a unique ID so clients can track pauses vs. final end.

**Lines 799–817 — call `_process_message`, publish result:** the response `OutboundMessage` is published. For WebSocket, a `_turn_end` sentinel follows. For WebUI sessions, a background task generates a session title.

**Lines 837–862 — cancellation handling:** if cancelled (e.g. by `/stop`), any checkpoint saved mid-turn is materialized into session history before re-raising `CancelledError`. This preserves tool results that completed before the cancel.

**Lines 869–887 — finally block:** removes the pending queue. Any messages left in it are re-published to the bus as fresh inbound messages so they aren't lost.

---

## Lifecycle (lines 889–909)

**`close_mcp` (lines 889–899):** drains all pending background tasks, then closes all MCP server connections via their `AsyncExitStack`. Called on shutdown.

**`_schedule_background` (lines 901–905):** wraps a coroutine in an `asyncio.Task` and tracks it in `_background_tasks`. Used for title generation and background consolidation.

**`stop` (lines 907–909):** sets `_running = False`; the `run()` loop exits on its next iteration.

---

## `_process_message()` (lines 912–1180)

Where session context is assembled and the runner is invoked. Two distinct paths.

### System/subagent path (lines 924–1013)

`msg.channel == "system"` means an internal message — typically a subagent announcing its result. Channel and `chat_id` are parsed from the system message's `chat_id` field.

Steps: load/create session → restore crash checkpoint and pending user turn → auto-compact → consolidate → persist subagent result into history *before* building the prompt (so it survives a subsequent crash) → build messages → call `_run_agent_loop` → save turn, clear checkpoints, save session → return `OutboundMessage` addressed to the originating channel.

### Normal user message path (lines 1015–1180)

1. **Media extraction** (lines 1017–1019): document text extracted from attachments so all channels share a unified representation.
2. **Session setup** (lines 1025–1033): load/create session, restore checkpoint and pending user turn, run auto-compact.
3. **Slash command dispatch** (lines 1036–1038): handle and return early if matched.
4. **Consolidation** (lines 1040–1043): trim history if over the token budget.
5. **Tool context + turn start** (lines 1045–1051): point routing-aware tools at this channel/chat; notify `MessageTool` that a new turn is starting.
6. **History load** (lines 1053–1058): load recent session history within the token replay budget.
7. **Prompt assembly** (lines 1060–1077): if a pending `ask_user` turn exists, inject the user's reply as a tool result; otherwise call `context.build_messages` normally.
8. **Progress and retry callbacks** (lines 1079–1109): `_bus_progress` publishes progress events with `_progress=True` metadata; `_on_retry_wait` publishes a retry-wait notice.
9. **Early user message persistence** (lines 1115–1124): save the user message to disk *before* calling the runner, then set `PENDING_USER_TURN_KEY`. A crash now won't silently lose the user's input.
10. **Call `_run_agent_loop`** (lines 1126–1139).
11. **Post-turn persistence** (lines 1145–1150): save new messages, clear crash flags, save session, schedule background consolidation.
12. **`MessageTool` suppression** (lines 1159–1161): if the `message` tool already sent something mid-turn, suppress the final response to avoid a duplicate — unless follow-up messages were injected.
13. **Build and return `OutboundMessage`** (lines 1166–1180): attach buttons for `ask_user` turns; mark `_streamed` in metadata if streaming was active.

---

## Session persistence (lines 1182–1289)

**`_sanitize_persisted_blocks` (lines 1182–1220):** strips volatile content from multimodal message blocks before writing to history. Drops runtime-context text blocks, replaces inline base64 image data with a placeholder path reference, and optionally truncates long text blocks.

**`_save_turn` (lines 1222–1265):** writes messages from the runner's output into session history. `skip` is the count of messages at the front already in history. Tool results are truncated and sanitized; user messages have the ephemeral runtime context block stripped; empty assistant messages are skipped entirely. A timestamp is added to every message if missing.

**`_persist_subagent_followup` (lines 1267–1289):** persists a subagent result message directly into session history before prompt assembly. Guards against duplicates by checking `subagent_task_id`. Returns `True` if a new entry was appended.

---

## Crash recovery (lines 1291–1390)

The loop has two crash guards that protect session state across restarts.

**Runtime checkpoint** (lines 1291–1304): `_set_runtime_checkpoint` saves the in-flight turn state (assistant message, completed tool results, still-running tool calls) into `session.metadata` and persists to disk after each tool round. `_clear_runtime_checkpoint` removes it on clean completion.

**`_checkpoint_message_key` (lines 1306–1316):** static helper returning a tuple of identifying fields from a message. Used for dedup comparison during restore.

**`_restore_runtime_checkpoint` (lines 1318–1370):** called at the start of every `_process_message`. If a checkpoint exists, it rebuilds the assistant message and completed tool results; tool calls that never completed get a synthetic error result. It then compares the rebuilt messages against the session tail to avoid double-writing messages already persisted. Returns `True` if anything was restored.

**Pending user turn** (lines 1296–1300, 1372–1390): `_mark_pending_user_turn` sets a flag right after the user message is persisted early; `_clear_pending_user_turn` clears it on clean completion. `_restore_pending_user_turn` (lines 1372–1390): if the flag is set and the last session message is a user message with no assistant reply, appends an error assistant message to close the turn so the session isn't left in an invalid state.

---

## `process_direct` (lines 1392–1415)

Public entry point used by the Python SDK and the CLI. Wraps a content string in an `InboundMessage` and calls `_process_message` directly — bypassing the bus and task system. No pending queue, no streaming segments.
