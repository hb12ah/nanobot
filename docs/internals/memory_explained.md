# Memory deep-dive (`nanobot/agent/memory.py`)

## TL;DR

Nanobot has four layers of memory:

```
user message arrives
    │
    ▼
Session (workspace/sessions/<key>.jsonl)
    │  per-turn append of every message; persisted atomically after each turn
    │  get_history() slices the unconsolidated tail for prompt assembly
    │
    │  overflow: prompt too     │  idle > idleCompactAfterMinutes
    │  large for context window │  (opt-in, default disabled)
    ▼                           ▼
Consolidator.archive()       AutoCompact._archive()
    │  LLM summarizes            │  LLM summarizes all unconsolidated
    │  evicted chunk             │  messages except last 8; wipes session
    │                            │  to just those 8 + summary
    └──────────┬─────────────────┘
               │  appends one entry to history.jsonl
               ▼
history.jsonl                   ← staging buffer; capped at 1000 entries
    │
    │  every ~2 h (cron)
    ▼
Dream.run()                     ← two-phase LLM job
    Phase 1: plain LLM call     → analysis of history batch + current files
    Phase 2: AgentRunner        → edits MEMORY.md / SOUL.md / USER.md
             (file tools)         may create workspace skills/*/SKILL.md
    │
    ▼
MEMORY.md   SOUL.md   USER.md   ← long-term stores; git auto-committed
    │
    │  every turn, at prompt assembly time
    ▼
context.build_messages()        ← injected into system prompt
```

The agent can also write directly to the long-term files mid-conversation via `MyTool` (`update_memory`, `update_soul`, `update_user`), bypassing the Consolidator / Dream pipeline entirely.

---

## Session — short-term memory (`session/manager.py`)

`Session` is the live conversation log for one `channel:chat_id`. It is the first and fastest layer of memory — every message is appended to it immediately, and it is what the LLM actually reads during prompt assembly.

**Storage format:** Each session is a JSONL file at `workspace/sessions/<key>.jsonl`. The first line is a metadata record (`_type: metadata`) carrying `key`, `created_at`, `updated_at`, `last_consolidated`, and the `metadata` dict (used for crash checkpoints, last summary, title, etc.). Every subsequent line is one message dict. Writes are atomic: the file is written to a `.tmp` sibling and then `os.replace`d over the real path.

**In-memory cache:** `SessionManager` keeps a `_cache` dict of loaded sessions. A session is loaded from disk on first access and stays in memory for the lifetime of the process. `invalidate` drops the cache entry (used by AutoCompact before reloading a fresh copy).

**`last_consolidated`:** An integer index into `session.messages`. Everything below it has already been summarized into `history.jsonl` by Consolidator or AutoCompact. `get_history()` only returns `messages[last_consolidated:]` — the unconsolidated tail — so evicted messages never re-enter the prompt.

**`get_history()`:** Slices the unconsolidated tail by message count (default 120) and optionally by a token budget from the tail. It enforces two legality rules: the slice must start at a user turn (not mid-tool-round), and orphan tool results at the front are dropped. Timestamps are optionally annotated on user turns so the LLM can reason about relative time.

**File cap:** `enforce_file_cap` (called during session save in loop.py) hard-limits the JSONL file to 2000 messages. If the file would exceed this, the oldest unconsolidated messages are raw-archived to `history.jsonl` and then dropped from the session. This is a last-resort guard against unbounded file growth and bypasses the LLM summarization step.

---

## MemoryStore — pure file I/O (lines 39–426)

`MemoryStore` is a thin wrapper with no business logic. It owns the paths and provides typed read/write methods. Everything else calls through it.

| File | Path | What it holds |
|------|------|--------------|
| `MEMORY.md` | `workspace/memory/MEMORY.md` | Long-term facts accumulated across conversations |
| `SOUL.md` | `workspace/SOUL.md` | Agent identity and personality |
| `USER.md` | `workspace/USER.md` | What the agent knows about the user |
| `history.jsonl` | `workspace/memory/history.jsonl` | Append-only log of Consolidator summaries |

Each entry in `history.jsonl` is a JSON line: `{"cursor": N, "timestamp": "...", "content": "..."}`. The cursor is a monotonically incrementing integer written by `_next_cursor` and mirrored to a `.cursor` file for fast reads.

A second cursor file, `.dream_cursor`, tracks the highest entry Dream has already processed — so Dream only picks up new entries since its last run.

`get_memory_context` (line 227) is called by `context.py` every turn to inject the current `MEMORY.md` into the system prompt.

---

## Consolidator — session overflow handler (lines 442–691)

`Consolidator` runs at the start of each turn (inside `_process_message`) when the session history is approaching the model's context window limit. Its job is to shrink the live session before the LLM call, without simply discarding history.

**How it decides to act:** `maybe_consolidate_by_tokens` estimates the token count of the full prompt (system prompt + history + tools). If it exceeds the input budget (`context_window - max_completion_tokens - 1024`), it enters a loop:

1. `pick_consolidation_boundary` — walks the session messages and finds a user-turn boundary that covers enough tokens to bring the prompt back under the target (50% of budget by default).
2. `archive` — calls the LLM with the chunk of evicted messages and a consolidation prompt template. The LLM writes a prose summary; `append_history` appends it to `history.jsonl`. If the LLM fails, `raw_archive` dumps the raw formatted messages instead (a degraded but lossless fallback).
3. `session.last_consolidated` is advanced to the boundary index and the session is saved. The evicted messages remain in the session object for the current turn but will not be included in future history loads.

The loop repeats up to 5 rounds per turn until the prompt fits. On success it stores the last summary in `session.metadata["_last_summary"]` so the next turn can inject a brief "here is what happened before" note into the prompt.

---

## Dream — background memory processor (lines 706–1008)

`Dream` is the heavyweight job that turns the `history.jsonl` staging buffer into durable edits to `MEMORY.md`, `SOUL.md`, and `USER.md`. It runs on a cron schedule (default every 2 hours) and is the only component that can create new workspace skill files.

### Entry point: `Dream.run()` (line 856)

```
read_unprocessed_history(since_cursor=last_dream_cursor)
    │
    │  slice to max_batch_size (20)
    ▼
Phase 1: plain LLM call
    input:  history batch text
            + current MEMORY.md (with per-line age annotations)
            + current SOUL.md
            + current USER.md
    output: analysis text (what to add, update, or remove)
    │
    ▼
Phase 2: AgentRunner with file tools
    input:  Phase 1 analysis
            + same file previews
            + list of existing skills (for dedup)
    tools:  read_file, edit_file, write_file (write restricted to skills/)
    output: targeted edits to MEMORY.md / SOUL.md / USER.md;
            new SKILL.md files under workspace/skills/
    │
    ▼
advance dream cursor (only on stop_reason == "completed")
compact_history() → trim history.jsonl to 1000 entries
git auto-commit if any files changed
```

### Phase 1 — analysis

A plain `chat_with_retry` call (no tools). The system prompt is rendered from `agent/dream_phase1.md`. The user message contains the history batch text plus previews of the three long-term files.

Before passing `MEMORY.md` to the LLM, `_annotate_with_ages` (line 810) appends per-line age suffixes using git blame — e.g. `some fact  ← 30d` — so the LLM can identify stale facts without having to infer it from content alone. Lines newer than `_STALE_THRESHOLD_DAYS` (14 days) get no suffix. If git is unavailable or the line count doesn't match, the raw file is passed unchanged.

Phase 1 produces a plain text analysis: what new facts to record, what old ones to update or delete, whether SOUL.md or USER.md need updating.

### Phase 2 — file editing via AgentRunner

Phase 2 delegates to `AgentRunner.run()` with a minimal tool registry: `read_file`, `edit_file`, and `write_file` (the last restricted to `workspace/skills/` so Dream can create skill files but can't write anywhere else). The system prompt is rendered from `agent/dream_phase2.md`; the user message is the Phase 1 analysis plus the same file previews and an existing-skills list for dedup.

The runner runs the standard LLM/tool loop. The LLM reads files it wants to inspect in full, then makes targeted `edit_file` calls rather than rewriting entire files — this minimizes git diff noise and reduces the chance of accidentally losing content.

### Cursor and compaction

The dream cursor is advanced **only** if Phase 2 completes with `stop_reason == "completed"`. A failed or partial run leaves the cursor where it was, so the same entries are retried on the next cron cycle.

`compact_history()` runs unconditionally after Phase 2, trimming `history.jsonl` to the 1000 most recent entries by count. The two mechanisms do not coordinate: if `history.jsonl` grows beyond 1000 entries between Dream runs, `compact_history` will silently drop entries the dream cursor hasn't reached yet. Those entries are lost without being distilled. This is a deliberate recency-over-completeness trade-off for the pathological case where Dream falls far behind; in normal operation the file stays well under the cap.

---

## AutoCompact — idle session compaction (`autocompact.py`)

AutoCompact is **disabled by default**. It activates only when `idleCompactAfterMinutes` is set in config (config key also accepted as `sessionTtlMinutes`). There is no hardcoded default TTL.

**Motivation:** Consolidator reacts to overflow *during a turn* — the user waits while LLM summarization calls happen before the response begins. AutoCompact moves that work to idle time so the next turn starts with an already-lean session at zero latency cost.

**When it runs:** `check_expired` is called from `run()` on every 1-second bus poll timeout (whenever no message arrives). It iterates all known sessions and checks whether `updated_at` is older than the TTL. Sessions with an active task are skipped (`key in active_session_keys`). Eligible sessions are scheduled as background tasks via `schedule_background(_archive(key))`.

**What `_archive` does:**

1. Reloads the session from disk (`sessions.invalidate` + `get_or_create`) to get the freshest state.
2. `_split_unconsolidated` splits the unconsolidated tail of session history into two parts: everything except the last 8 messages goes to `archive_msgs`; the last 8 are kept as `kept_msgs`. The 8-message suffix is always retained to preserve immediate context for when the user returns.
3. `consolidator.archive(archive_msgs)` — the same LLM summarization call Consolidator uses during overflow — appends a summary entry to `history.jsonl`.
4. The live session is replaced with just `kept_msgs`, `last_consolidated` reset to 0, and saved to disk.
5. The summary and `last_active` timestamp are stored in `self._summaries[key]` (in-memory) and in `session.metadata["_last_summary"]` (on disk, for process restarts).

**On the next turn:** `prepare_session` is called at the start of `_process_message`. It checks `_summaries` (fast path, same process) or `session.metadata["_last_summary"]` (after a restart) and returns a formatted context string like `"Inactive for 47 minutes. Previous conversation summary: ..."` that gets injected into the prompt — so the agent knows it was away and what was discussed.

**Difference from Consolidator:** Consolidator evicts the minimum necessary to fit within the context window. AutoCompact is more aggressive: it archives everything except the last 8 messages regardless of whether the session was near its limit, trading context richness for guaranteed low latency on the next turn.
