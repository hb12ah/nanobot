# Prompt Caching Explained

## What is prompt caching?

Every time nanobot calls the LLM, it sends the full input context: system prompt, tool schemas, and the entire conversation history. Encoding all those tokens from scratch on every turn is expensive and slow.

Anthropic's prompt caching lets you mark a **prefix boundary** in the input. On the first request, Anthropic encodes the prefix normally and stores the resulting KV cache state server-side. On subsequent requests, the model checks whether it has seen this prefix before — if the prefix is byte-identical, it skips re-encoding those tokens and reads the stored KV state directly. Cache reads cost roughly 10% of normal input token price.

Placing a cache marker is an instruction to the model: check whether this prefix has been seen before. On a hit, those tokens are not re-processed — the model resumes directly from the stored KV state. On a miss, the prefix is processed normally and the result is stored for future requests.

The marker is a `cache_control` field attached to a content block:

```json
{ "type": "text", "text": "...", "cache_control": { "type": "ephemeral" } }
```

`ephemeral` means the cache has a 5-minute TTL. It is best-effort — Anthropic may evict it early under load.

## What gets marked

`_apply_cache_control()` in `anthropic_provider.py` marks three things before every request:

```
[system prompt]  ← cache_control on last block
[tools]          ← cache_control on last tool
[conversation]   ← cache_control on messages[-2]  (previous assistant reply)
[messages[-1]]   ← NOT marked (current user input, changes every turn)
```

Each marker says: **cache everything from the start up to and including this point.** It is a prefix boundary, not a per-block flag. One marker covers all prior content in one shot.

## How the conversation cache grows

With three messages in flight (the minimum for the `len(new_msgs) >= 3` guard to fire), the cache boundary slides forward two messages each turn:

```
Turn 1  →  [sys][tools][u₁ a₁*]  ·  [u₂]         marker on a₁
Turn 2  →  [sys][tools][u₁ a₁ u₂ a₂*]  ·  [u₃]   marker on a₂, hit on [sys][tools][u₁ a₁]
Turn 3  →  [sys][tools][u₁ a₁ u₂ a₂ u₃ a₃*]  ·  [u₄]   hit on [sys][tools][u₁ a₁ u₂ a₂]
```

The prefix grows by two messages per turn (one user + one assistant). Only the current user input falls outside the cache — it cannot be cached now because it is what the model is responding to, and caching it would leave nothing for the model to generate from. It will be inside the cached prefix on the next turn.

## Why tool schemas need stable ordering

Caching requires the prefix to be **byte-identical** across requests. If any tool schema changed position, the tool block prefix would differ and every turn would be a cache miss.

`ToolRegistry.get_definitions()` guarantees a stable sort: built-in tools alphabetically first, then MCP tools alphabetically. The result is also memoized and only invalidated when a tool is registered or unregistered (which only happens at startup). This makes the tool block a reliable cache anchor.

## What breaks the cache

Any modification to history that changes a byte before the marker causes a full cache miss for that prefix:

- **`_microcompact()`** — replaces old tool result strings with `"[tool result omitted]"`. Any compacted message is now different from what was cached.
- **`_snip_history()`** — drops old messages to stay within the context window. The prefix is now shorter and mismatched.
- **`_backfill_missing_tool_results()`** — inserts synthetic error results for orphaned tool calls, shifting message positions.

All three run before `_apply_cache_control()` on every turn. In a short, clean conversation the cache works well. In a long session with heavy tool use, compaction and snipping are likely, and the conversation cache will frequently miss — though the system prompt and tool schema caches are unaffected by message history changes and remain stable.

## The three cache layers summarised

| Layer | Stability | When it misses |
|---|---|---|
| System prompt | Very stable — fixed at session start | Never, unless system prompt changes |
| Tool schemas | Stable — sorted, memoized, startup-only | Never mid-session |
| Conversation history | Grows each turn, fragile | Any compaction, snip, or injection |
