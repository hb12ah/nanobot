# AgentRunner deep-dive (`nanobot/agent/runner.py`)

## TL;DR

`AgentRunner` is the inner LLM loop. Given a list of messages and a tool registry, it repeatedly: (1) repairs and compresses history, (2) calls the LLM, (3) runs any requested tools, (4) feeds results back — until the model produces a final text answer or a stop condition is hit. It knows nothing about users, sessions, or channels; that's `AgentLoop`'s job.

The main entry point is `run(spec)` (lines 234–567). Everything else is a helper it calls. The two trickiest concepts are **injection/drain** (queuing follow-up user messages that arrive mid-turn) and **hook** (lifecycle callbacks that let the caller observe or extend the loop without touching runner code).

---

## What `AgentRunner` is (lines 100–104)

`AgentRunner` is the **inner engine** — it knows nothing about channels, sessions, or users. It takes a fully-assembled list of messages, calls the LLM, executes tools, and repeats until there's a final text answer. The outer `AgentLoop` handles all product concerns; it just calls `runner.run(spec)`.

---

## Module-level constants (lines 39–52)

Tuning knobs defined as module constants. Notable ones:

| Constant | Meaning |
|----------|---------|
| `_MAX_EMPTY_RETRIES = 2` | How many times to retry before giving up on an empty LLM response |
| `_MAX_LENGTH_RECOVERIES = 3` | How many times to continue a response that was cut off mid-sentence |
| `_MAX_INJECTIONS_PER_TURN = 3` | Max queued follow-up messages consumed in one turn |
| `_MAX_INJECTION_CYCLES = 5` | Max times the loop can re-enter due to injections |
| `_COMPACTABLE_TOOLS` | Tools whose old results get compressed to save context space |
| `_BACKFILL_CONTENT` | The placeholder inserted when a tool result is missing |

---

## Data classes (lines 56–97)

**`AgentRunSpec`** (lines 56–84) — everything the runner needs to do its job: the messages to start from, the tool registry, the model name, limits, callbacks, and optional features. Think of it as the runner's input configuration.

**`AgentRunResult`** (lines 86–97) — what the runner returns: the final text, the full message list (including all tool rounds), token usage, which tools were called, and why the run stopped.

---

## `run()` — the main loop (lines 234–567)

### Initialisation (lines 235–249)

Sets up local state for the entire run: working copy of `messages`, counters for retries and injection cycles, accumulators for usage and tool events, and the `hook` object (explained at the end).

### The `for` loop — one iteration = one LLM call (lines 251–532)

---

#### Step 1 — Context governance (lines 252–275)

Before calling the LLM, the history is preprocessed into a temporary copy called `messages_for_model`. The real `messages` list is never touched here — governance only affects what gets sent to the model this turn.

| Helper called | What it does |
|---------------|-------------|
| `_drop_orphan_tool_results` | Remove tool result messages that have no matching tool call above them |
| `_backfill_missing_tool_results` | Insert a synthetic error result for any tool call that has no result below it |
| `_microcompact` | Replace old bulky tool results with `[result omitted]` summaries |
| `_apply_tool_result_budget` | Truncate any result that exceeds the per-result character limit |
| `_snip_history` | Drop old messages from the front if the history is too long for the context window |

The drop+backfill pair is run twice (lines 257–258 and 263–264): once before snipping, and once after, because snipping can create new orphans.

If governance crashes entirely (lines 265–275), a fallback applies just the two essential repairs (drop + backfill) on the unmodified `messages`.

---

#### Step 2 — Fire the hook, call the LLM (lines 276–283)

1. A `context` object is created (line 276) — a snapshot of the current iteration's state that gets passed to all hook callbacks.
2. `hook.before_iteration(context)` fires (line 277), giving observers a chance to act before the LLM is called.
3. `_request_model` is called (line 278) — the actual LLM call with timeout, streaming, and retry handling.
4. Usage stats from this call are accumulated into the running totals (lines 279–283).

---

#### Step 3a — LLM wants tool calls (lines 285–389)

Entered when `response.should_execute_tools` is true.

**Lines 286–290:** Build the list of tool calls. If `ask_user` appears anywhere in the list, truncate there — the agent has a question and should not run any more tools until the user answers.

**Lines 291–292:** If streaming is active, signal stream-end with `resuming=True` (the spinner should keep going, tools are about to run).

**Lines 294–312:** Build the assistant message (including tool call declarations) and append it to `messages`. Emit a `"awaiting_tools"` checkpoint so a crash at this point can be recovered.

**Line 314:** `hook.before_execute_tools(context)` — lets observers log tool calls, emit progress events, etc.

**Lines 316–324:** Call `_execute_tools`, which runs all the tool calls and returns three lists: results, event summaries, and whether there was a fatal error.

**Lines 325–341:** Convert each result into a `role: tool` message and append it to `messages`. The `ask_user` result is skipped — it's handled separately as a clean stop.

**Lines 342–367:** Handle fatal errors:
- `AskUserInterrupt` (lines 343–351) — the model asked the user something. Set `stop_reason = "ask_user"`, signal stream-end, and `break`.
- Any other fatal error (lines 352–367) — record it as `stop_reason = "tool_error"`, try to drain injections, then `break` (or `continue` if injections were found).

**Lines 368–389:** Happy path — emit a `"tools_completed"` checkpoint, reset the empty/length retry counters, drain any injections that arrived while tools were running, call `hook.after_iteration`, and `continue` to the next iteration.

---

#### Step 3b — LLM returns text (lines 391–532)

**Lines 391–396:** Safety check — if the LLM returned tool calls but with a finish reason that doesn't allow execution (e.g. `stop` instead of `tool_calls`), log a warning and ignore them.

**Lines 398–428 — Empty response handling:**
`hook.finalize_content` post-processes the text (e.g. strips `<think>` blocks). If the result is blank:
- Up to `_MAX_EMPTY_RETRIES`: log a warning, signal stream-end, and retry the iteration.
- After all retries exhausted: call `_request_finalization_retry` (lines 421–428), which sends the same history plus an explicit "please write a final response" message. No tools are offered this time.

**Lines 430–449 — Length recovery:**
If `finish_reason == "length"` the model ran out of output tokens mid-sentence. Append what it wrote, then append a "please continue" message (`build_length_recovery_message`), signal stream-end with `resuming=True`, and loop. Up to `_MAX_LENGTH_RECOVERIES` times.

**Lines 451–475 — Injection drain before stream-end:**
Before signalling stream-end to the outside world, check whether any follow-up messages arrived while the model was responding. If yes, `should_continue = True`, which keeps the stream alive (`resuming=True`) so the streaming channel doesn't prematurely close the response card.

**Lines 477–493 — LLM error case:**
`finish_reason == "error"`. Set `stop_reason = "error"`, insert a placeholder assistant message, try to drain injections (if any arrived, loop instead of stopping), then `break`.

**Lines 494–510 — Empty final response case:**
After all retries, the response is still blank. Set `stop_reason = "empty_final_response"`, insert a fallback message, try to drain injections, then `break`.

**Lines 512–532 — Success:**
Append the assistant message and emit a `"final_response"` checkpoint. Set `final_content = clean`, call `hook.after_iteration`, and `break`.

---

### `for`-`else` — max iterations hit (lines 533–556)

Python's `for-else` runs only if the loop exhausted all iterations without a `break`. This means the agent never produced a final response. Sets `stop_reason = "max_iterations"`, appends a "I've run out of steps" message, and drains any remaining injections (so they don't bounce back as new inbound messages).

### Return (lines 558–567)

Packages everything accumulated during the run into an `AgentRunResult` and returns it to the caller.

---

## Helper methods

### LLM call helpers

**`_build_request_kwargs` (lines 569–589)**
Assembles the keyword arguments for the LLM provider call: messages, tools, model name, retry mode, and optional temperature/max\_tokens/reasoning\_effort.

**`_request_model` (lines 591–665)**
The actual LLM call. Handles three modes:
- **Streaming** (`hook.wants_streaming()`) — calls `chat_stream_with_retry` and feeds each chunk to `hook.on_stream`.
- **Progress streaming** — like streaming but pipes deltas to `spec.progress_callback` instead of the hook. Used when a non-streaming channel still wants incremental progress events.
- **Non-streaming** — calls `chat_with_retry`.

In all modes, applies a timeout (default 300 s, configurable via `NANOBOT_LLM_TIMEOUT_S`). A timeout returns a synthetic `LLMResponse` with `finish_reason="error"` instead of raising.

**`_request_finalization_retry` (lines 667–675)**
Appends a "please write a final response now" system message and calls the LLM without tools. Used after all empty-response retries are exhausted.

### Usage accounting (lines 677–699)

Three small static helpers:
- `_usage_dict` — converts raw provider usage to a clean `dict[str, int]`.
- `_accumulate_usage` — adds one usage dict into a running total.
- `_merge_usage` — merges two usage dicts into a new one (used after a finalization retry).

### Tool execution

**`_execute_tools` (lines 701–740)**
Orchestrates running all the tool calls from one LLM response. Delegates to `_partition_tool_batches` to decide which can run concurrently. Runs concurrent batches with `asyncio.gather`; sequential ones one at a time. Stops a batch early if `AskUserInterrupt` is seen. Collects all results, events, and the first fatal error.

**`_run_tool` (lines 742–845)**
Runs a single tool call. The steps are:
1. Check `repeated_external_lookup_error` — if the model has tried the same web/external lookup too many times, block it immediately.
2. Call `prepare_call` on the registry — validates parameters and pre-resolves the tool object.
3. Execute the tool.
4. On exception: check `_classify_violation` first; if not a policy violation, return the error text as a soft result (or fatal, if `fail_on_tool_error`).
5. If the result string starts with `"Error"`: same violation check, same soft/fatal logic.
6. On success: build a brief `detail` summary (≤120 chars) for the event log and return.

**`_partition_tool_batches` (lines 1179–1202)**
Splits tool calls into batches for `_execute_tools`. Tools marked `concurrency_safe` are grouped together; non-safe tools get their own single-item batch. This means read-only tools can run in parallel while writes and interactive tools remain sequential.

### Safety boundary helpers (lines 847–937)

**SSRF** (Server-Side Request Forgery): the model tries to fetch an internal/private URL. Detected by string markers in the error text. Returned as a non-retryable error with a firm message telling the model not to try again through any route.

**Workspace violations**: the model tries to access a path outside the allowed directory (or a path traversal). First occurrence returns a soft hint. Repeated attempts (`repeated_workspace_violation_error`) escalate to a stronger message.

`_classify_violation` (lines 890–928) — checks both categories and returns a formatted error tuple, or `None` if neither applies (letting the caller handle it normally).

`_ssrf_soft_payload` / `_event_detail` (lines 930–937) — small formatting helpers for the above.

### Checkpointing (lines 939–946)

**`_emit_checkpoint`** — calls `spec.checkpoint_callback` with the current turn state. The callback is supplied by `AgentLoop` and saves the state to session metadata, so a crash mid-turn can be recovered next time.

### Message append helpers (lines 948–967)

**`_append_final_message`** — appends (or updates) the last assistant message. If the last message is already an assistant message without tool calls, it updates it in place to avoid a duplicate.

**`_append_model_error_placeholder`** — appends a `[Assistant reply unavailable]` message when the LLM returned an error, so the history stays well-formed.

### Tool result normalization (lines 969–994)

**`_normalize_tool_result`** — two things:
1. Calls `maybe_persist_tool_result`: if the result is very large, write it to disk and return a reference path instead of the raw text. This keeps the in-memory message list lean.
2. Truncates anything that still exceeds `max_tool_result_chars`.

### Context governance helpers

**`_drop_orphan_tool_results` (lines 996–1020)** — walks the message list, tracks which tool call IDs the assistant has declared, and removes any `role: tool` message whose ID isn't in that set.

**`_backfill_missing_tool_results` (lines 1022–1061)** — the reverse: finds tool call IDs declared by the assistant that have no corresponding result, and inserts a synthetic error result immediately after the assistant message.

**`_microcompact` (lines 1063–1087)** — keeps only the 10 most recent results from compactable tools. Older ones are replaced with `[tool result omitted from context]`. Only applies to results longer than 500 characters.

**`_apply_tool_result_budget` (lines 1089–1108)** — passes every tool result through `_normalize_tool_result` again, ensuring nothing exceeds the per-result character budget.

**`_snip_history` (lines 1110–1177)** — estimates the token count of the full message list. If it exceeds the context window budget, drops messages from the front (keeping system messages). Always ensures the kept window starts with a user message — some providers reject a history that starts with an assistant turn.

### Injection and Drain (lines 106–232)

**The problem:** A user sends a follow-up message *while* the agent is still processing their first one. In a naive system this would either get lost or spawn a competing task that interleaves chaotically.

**The solution:** A `pending_queue` (an `asyncio.Queue`) sits alongside each active session. New messages that arrive mid-turn get routed into this queue instead of starting a new task.

**"Injection"** = feeding one of those queued messages into the *current* turn's message list, so the agent can respond to it without ending the turn.

**"Drain"** = pulling items out of the queue. `_drain_injections` calls the `injection_callback` (which reads from the `pending_queue`) and returns them as normalized `role: user` messages.

`_try_drain_injections` (lines 145–187) is the higher-level wrapper — it calls `_drain_injections` and, if messages came back, appends them and returns `should_continue=True` so the for loop keeps going. It checks in three places:

1. **After tool execution** (line 382) — before the next LLM call, in case a follow-up arrived while tools were running.
2. **After the final response** (line 462) — the most common case: the agent is done, but another message snuck in right at the end.
3. **After errors** (lines 360, 486, 503) — so error recoveries don't silently drop a queued message.

`injection_cycles` is a counter to prevent an infinite loop: if injections keep arriving endlessly, `_MAX_INJECTION_CYCLES = 5` forces the loop to end.

`_drain_injections` (lines 189–232) handles the mechanics: calls the callback, normalises each item into a `role: user` dict, and caps the batch at `_MAX_INJECTIONS_PER_TURN`.

`_append_injected_messages` (lines 123–143) appends the drained messages while preserving role alternation: if the last message in history is already a `user` message, it merges the injection content into it rather than creating a second consecutive user message (which many providers reject).

---

## What `hook` is doing (lines 15, 235, 276–531)

Think of `hook` as a **set of notification slots** bolted onto the runner. The runner itself doesn't know anything about channels, streaming, progress bars, or logging — that's not its job. But the code that *calls* the runner often needs to react to things happening inside it: show a spinner while tools run, stream text to the user as it arrives, log which tools were used, etc.

The `AgentHook` class (defined in `hook.py`) defines six callback points:

| Callback | When it fires |
|----------|--------------|
| `before_iteration(context)` | At the start of every iteration, before the LLM is called |
| `on_stream(context, delta)` | Each time a chunk of text arrives from the LLM (streaming only) |
| `on_stream_end(context, resuming)` | When the LLM stops outputting text. `resuming=True` means tools follow; `resuming=False` means this is the final answer |
| `before_execute_tools(context)` | After the model requests tool calls, before they run |
| `after_iteration(context)` | At the end of every iteration |
| `finalize_content(context, content)` | A synchronous filter applied to the LLM's text output before it's used |

The `context` argument is an `AgentHookContext` — a mutable snapshot of everything the runner knows at that moment: iteration number, messages so far, the LLM response, tool calls, tool results, usage stats, final content, and stop reason. The hook reads from it and can write back to it (e.g. `context.streamed_content = True`).

The base `AgentHook` class does nothing — all methods are no-ops. The main loop injects its own `_LoopHook` subclass, which:
- On `before_execute_tools`: emits progress events to the channel and logs the tool calls.
- On `on_stream`: strips `<think>` blocks and forwards clean text to the streaming callback.
- On `on_stream_end`: calls the stream-end callback, resets the streaming buffer.
- On `after_iteration`: emits tool-finish progress events and logs token usage.
- On `finalize_content`: strips `<think>` blocks from the final text.

When multiple hooks need to run (e.g. the main loop hook plus a custom plugin hook), `CompositeHook` fans out each callback to all of them in order, catching and logging exceptions from non-critical hooks so a buggy plugin can't crash the runner.

The runner uses `hook.wants_streaming()` (lines 291, 409, 419, 440, 470) as a fast check before calling the streaming-specific methods, to avoid overhead when no one is listening.
