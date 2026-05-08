# Subagent deep-dive (`nanobot/agent/subagent.py`)

## TL;DR

```
User message
    │
    ▼
MessageBus.inbound
    │
    ▼
AgentLoop._dispatch()
    │  registers pending_queue for this session
    ▼
_process_message() → _run_agent_loop() → runner.run()   ← MAIN TURN (stays open)
    │
    │  LLM calls spawn_tool
    ▼
SubagentManager.spawn()
    │  records origin (channel, chat_id, session_key)
    │  creates SubagentStatus(task_id)
    ▼
asyncio.create_task(_run_subagent(...))   ← SUBAGENT runs concurrently on same event loop
    │
    │  main runner continues:
    │  LLM writes "I've started the task..." → iteration ends
    │  _drain_pending: queue empty + subagent still running → BLOCKS on pending_queue.get()
    │
    │  (subagent runs in background)
    │  _run_subagent: builds fresh ToolRegistry, calls runner.run() — no session, no injection_callback
    │  _SubagentHook: no streaming, only debug logging + SubagentStatus updates
    │
    ▼
_announce_result()
    │  publishes InboundMessage(channel="system", sender_id="subagent",
    │                           session_key_override=origin.session_key)
    ▼
MessageBus.inbound
    │
    │  AgentLoop sees session_key_override matches active pending_queue
    ▼
pending_queue.put_nowait(announce_msg)   ← routes back to main turn, NOT a new _dispatch
    │
    │  _drain_pending unblocks, returns announce_msg as injected user message
    ▼
main runner.run() continues
    │  LLM reads subagent result, writes final response
    ▼
runner.run() returns → main turn ends
    │
    │  finally: pending_queue popped; any leftover messages re-published to bus
    ▼
MessageBus.outbound → channel → user
```

**Key invariants:**
- The main turn stays open (runner.run() does not return) until all subagents spawned in it finish or the 300 s timeout expires.
- Subagents have no session, no pending queue, and no injection callback — they are stateless fire-and-forget `AgentRunner.run()` calls.
- The announcement message is routed by `session_key_override`, not by `chat_id`, so it lands in the right pending queue even for thread-scoped sessions.

---

## `SubagentStatus` — real-time tracking (lines 28–41)

`SubagentStatus` is a dataclass that records the live state of a running subagent. It is keyed by `task_id` in `SubagentManager._task_statuses` and updated by `_SubagentHook.after_iteration` on every LLM iteration. Fields:

| Field | Meaning |
|-------|---------|
| `task_id` | 8-char UUID prefix — the logical identifier for this subagent job |
| `label` | Short display label (first 30 chars of the task, or caller-supplied) |
| `phase` | Current lifecycle stage: `initializing → awaiting_tools → tools_completed → final_response → done / error` |
| `iteration` | How many LLM calls have completed |
| `tool_events` | `[{name, status, detail}, ...]` — one entry per tool call |
| `usage` | Token usage accumulated so far |
| `stop_reason` | Final stop reason from `AgentRunResult` (`completed`, `tool_error`, `error`, etc.) |

`_task_statuses` is cleaned up by the done-callback on the asyncio Task, so it only holds entries for running subagents.

---

## `_SubagentHook` — stripped-down hook (lines 44–68)

Subagents reuse `AgentRunner` but with a minimal hook that differs from `_LoopHook` in two critical ways:

1. **No streaming.** `wants_streaming()` is not overridden, so it inherits `AgentHook`'s default of `False`. The runner uses non-streaming `chat_with_retry` instead of `chat_stream_with_retry`. No content deltas are ever forwarded to the channel — the result reaches the user only via the announcement message.

2. **No injection callback.** `AgentRunSpec` for the subagent has no `injection_callback`. `_drain_injections` in the runner returns immediately on `None`. The subagent runner loop is entirely isolated from the message bus.

What `_SubagentHook` does provide:
- `before_execute_tools`: debug-logs each tool call with `task_id` for tracing.
- `after_iteration`: updates `SubagentStatus` (phase, iteration, tool_events, usage, error).

---

## `SubagentManager.spawn()` — creation (lines 112–152)

`spawn()` is called by `SpawnTool.execute()` during the main runner's tool-execution phase. Steps:

1. Generate `task_id = str(uuid.uuid4())[:8]` — short, human-readable, appears in logs and the reply message.
2. Capture `origin = {channel, chat_id, session_key}` from the spawning message. This is the return address used later by `_announce_result`.
3. Create and store a `SubagentStatus` in `_task_statuses[task_id]`.
4. `asyncio.create_task(_run_subagent(...))` — the task is scheduled on the event loop immediately but doesn't start running until the current coroutine yields.
5. Store the task in `_running_tasks[task_id]` and index it under `_session_tasks[session_key]` so `get_running_count_by_session` can answer "is anyone still running for this session?"
6. Attach a done-callback `_cleanup` that removes the task from all three dicts when it finishes.
7. Return a human-readable confirmation string to the LLM (`"Subagent [label] started (id: task_id)"`).

---

## `_run_subagent()` — execution (lines 154–254)

The coroutine that runs inside the asyncio Task. It is completely self-contained:

**Tool registry:** A fresh `ToolRegistry` is built from scratch — filesystem tools, optionally shell (`ExecTool`) and web tools. No `SendMessageTool`, no `SpawnTool` — subagents cannot message the user directly or spawn their own subagents. A separate `FileStates` instance is created so read-dedup state is isolated from the parent session.

**System prompt:** `_build_subagent_prompt()` renders a focused system prompt (template `agent/subagent_system.md`) containing only time context and the skills summary — no memory, no soul, no user file. The task string becomes the sole user message.

**Runner call:**
```python
result = await self.runner.run(AgentRunSpec(
    initial_messages=[system_prompt_msg, task_msg],
    tools=tools,
    hook=_SubagentHook(task_id, status),
    injection_callback=None,   # no bus access
    fail_on_tool_error=True,   # any tool error is fatal
    ...
))
```

`fail_on_tool_error=True` means the first tool error terminates the run with `stop_reason="tool_error"` rather than letting the LLM retry. This is stricter than the main loop.

**Result dispatch:** After `runner.run()` returns, the stop reason determines which `_announce_result` path is taken:
- `tool_error` → partial progress summary (last 3 successful tool events + the failure).
- `error` → `result.error` string.
- anything else → `result.final_content`.

---

## `_announce_result()` — routing back (lines 256–299)

Publishes one `InboundMessage` to `MessageBus.inbound`:

```python
InboundMessage(
    channel="system",
    sender_id="subagent",
    chat_id=f"{origin['channel']}:{origin['chat_id']}",
    session_key_override=origin.session_key,   # ← routes to spawning session
    metadata={"injected_event": "subagent_result", "subagent_task_id": task_id},
    content=rendered_announcement,
)
```

`session_key_override` is the critical field. The main loop's bus-routing code checks whether the message's session key matches an active `pending_queue`. Because `session_key_override` is set to the spawning session's key, it matches and the message goes into `pending_queue.put_nowait()` — it is routed as a mid-turn injection rather than triggering a new `_dispatch`. The `task_id` in metadata lets the main loop (and the LLM prompt) identify which subagent is reporting back.

---

## `_drain_pending` and the 300 s wait — main turn lifecycle (loop.py lines 579–637)

`_drain_pending` is the `injection_callback` passed to the main runner. It is called by `_try_drain_injections` at the end of each runner iteration. Its logic:

1. **Non-blocking drain** (`get_nowait`): pull everything already in `pending_queue`. If items were found, return them immediately.
2. **Blocking wait**: if nothing was drained *and* `get_running_count_by_session > 0` (subagents still alive), block on `await asyncio.wait_for(pending_queue.get(), timeout=300)`.
3. **Timeout**: if 300 s elapse with no message, log a warning and return an empty list. The runner treats this like no injection and proceeds to end the turn if there's nothing else to do.

This is what keeps the main turn alive. The runner's for loop parks at `_try_drain_injections` until either the subagent result arrives or time runs out.

**Timeout consequence:** The subagent task is **not cancelled** on timeout. It continues running as an orphaned asyncio Task. When it eventually finishes, its announcement is published to the bus. If a new turn is active at that point, the message lands in the new turn's `pending_queue` and is processed out of context. If no turn is active, the `finally` block in `_dispatch` will re-publish it to the bus as a fresh inbound message, starting a new turn.

---

## Cancellation (lines 339–347)

`cancel_by_session(session_key)` is called when a session is torn down (e.g. `/cancel` command or session close). It finds all task IDs for the session in `_session_tasks`, cancels their asyncio Tasks, and awaits them with `return_exceptions=True` so a `CancelledError` doesn't propagate. The done-callback `_cleanup` then removes them from all dicts.
