# Nanobot Explained

## What it is

Nanobot is an async Python AI agent server. You install it, configure an LLM provider, then run one command to get a long-lived process that accepts messages from chat platforms (or a built-in web UI), runs them through an LLM with tools, and streams replies back.

---

## Starting up

```
pip install nanobot
nanobot onboard        # writes ~/.nanobot/config.json
nanobot gateway        # starts everything
```

`nanobot` is a CLI binary wired in `pyproject.toml`:

```
nanobot = "nanobot.cli.commands:app"   (Typer)
```

The call chain from terminal command to running server:

```
$ nanobot gateway
    ↓
pyproject.toml: nanobot = "nanobot.cli.commands:app"
    ↓
Typer routes "gateway" subcommand → gateway() [cli/commands.py:609]
    ↓
_run_gateway() [cli/commands.py:634]
    ↓ constructs: MessageBus, AgentLoop, CronService, ChannelManager
    ↓
asyncio.run(run())   ← blocks forever
```

---

## The three permanent coroutines

Inside `run()`, `asyncio.gather` keeps three coroutines alive indefinitely:

```
asyncio.gather(
    agent.run(),          ← message dispatch loop
    channels.start_all(), ← all channel adapters
    _health_server(...),  ← HTTP /health endpoint
)
```

---

## MessageBus: the only coupling

Channels and the AgentLoop never reference each other. Everything flows through `MessageBus`:

```
Channel adapter
    → bus.publish_inbound(InboundMessage)
        → AgentLoop.run() while-loop picks it up
            → asyncio.create_task(_dispatch(msg))
                → AgentRunner → LLM + tools
                    → bus.publish_outbound(OutboundMessage)
                        → Channel adapter sends reply
```

---

## AgentLoop (`agent/loop.py`)

The forever loop:

```python
while self._running:
    msg = await bus.consume_inbound()          # blocks until a message arrives
    asyncio.create_task(self._dispatch(msg))   # handle concurrently per session
```

Each session gets its own task so multiple users don't block each other. `AgentRunner` (`agent/runner.py`) does the inner LLM → tool-call → LLM iteration until the model stops or hits max iterations.

---

## Channels (`channels/`)

Each channel subclasses `BaseChannel`, which provides `_handle_message()` → `bus.publish_inbound()`. Channels are started concurrently by `ChannelManager.start_all()`.

### WebSocket channel → AgentLoop pipeline

The WebSocket channel is one of those adapters. It doubles as the web server for the built-in UI:

```
User opens browser
    ↓
WebSocketChannel.start() [channels/websocket.py:987]
    registers two hooks with websockets.serve():
        process_request → _dispatch_http()   ← all plain HTTP
        handler         → _connection_loop() ← WebSocket frames

    HTTP GET /
        → _dispatch_http → _serve_static()
        → reads nanobot/web/dist/index.html  ← pre-built React app
        → browser renders the UI

    React app opens WebSocket to same port
        → WS upgrade → _connection_loop() [websocket.py:1036]
            ↓ raw frame arrives
            _parse_inbound_payload() / _handle_envelope() [websocket.py:1078, 1166]
            ↓ typed message envelope (new_chat / attach / message)
            BaseChannel._handle_message() [channels/base.py:146]
            ↓
            bus.publish_inbound(InboundMessage)  ← onto the shared queue
                ↓
            ←── MessageBus inbound queue ───────────────────────┐
                                                                │
AgentLoop.run() [agent/loop.py:682]  ← created in _run_gateway()
    while self._running:
        msg = await bus.consume_inbound()   ← picks it up here
        asyncio.create_task(self._dispatch(msg))
            ↓
        AgentRunner → LLM → tools → OutboundMessage
            ↓
        bus.publish_outbound()
            ↓
        WebSocketChannel.send() streams reply back to browser
```

The static dist path (`nanobot/web/dist/`) is found by `ChannelManager` at startup:

```python
# channels/manager.py
static_path = _default_webui_dist()   # imports nanobot.web, finds dist/
kwargs["static_dist_path"] = static_path
channel = WebSocketChannel(section, bus, **kwargs)
```

---

## Cron service (`cron/service.py`)

A self-rescheduling asyncio timer loop — no external scheduler. On each tick it finds due jobs, runs them via `on_job()`, then re-arms itself to sleep until the next job.

At startup, one system job is registered: **Dream** — the background memory consolidation task that summarizes conversation history into `MEMORY.md` on a configurable interval (default 2h).

User jobs are added at runtime through natural language: the agent calls the `cron` tool (`action="add"`) when you ask it to schedule something. Jobs are persisted as JSON in `~/.nanobot/<workspace>/cron/jobs.json`.

When a user job fires, `on_job()` calls `agent.process_direct()` — the same LLM turn pipeline, just triggered by the timer instead of a user message. If `deliver=True`, the response is sent back to the channel/chat that was active when the job was created.

---

## Python SDK (`nanobot.py`)

`Nanobot.run(message)` is a single-turn façade over the same `AgentLoop.process_direct()`. The caller controls the loop; session history is preserved across calls via `session_key`. No channels, no cron, no gateway — just call and get a result.

---

## Key files at a glance

| File | Role |
|------|------|
| `cli/commands.py` | Entry point; wires all components and runs `asyncio.gather` |
| `bus/queue.py` | `MessageBus` — the only coupling between channels and agent |
| `agent/loop.py` | `AgentLoop` — forever `while` loop, dispatches per session |
| `agent/runner.py` | `AgentRunner` — inner LLM + tool-call iteration |
| `channels/websocket.py` | WebSocket server + static file server for the web UI |
| `channels/manager.py` | Discovers and starts all enabled channel adapters |
| `cron/service.py` | `CronService` — asyncio timer loop for scheduled jobs |
| `agent/memory.py` | `Dream` — background memory consolidation |
| `nanobot.py` | Single-turn Python SDK façade |
