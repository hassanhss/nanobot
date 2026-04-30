# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is an ultra-lightweight AI agent framework written in Python. It keeps a small, readable core agent loop while supporting chat channels, memory, MCP, and practical deployment paths. Think of it as a personal AI agent that can run in CLI, chat apps (Telegram, Discord, Slack, Feishu, WeChat, etc.), or as an API server.

## Build & Development Commands

```bash
# Install with dev dependencies
pip install -e ".[dev]"

# Run all tests
pytest

# Run a single test file
pytest tests/test_nanobot_facade.py

# Run a specific test
pytest tests/test_nanobot_facade.py::test_something -v

# Lint
ruff check nanobot/

# Format
ruff format nanobot/
```

## Architecture

The core flow is: **Channel → MessageBus → AgentLoop → AgentRunner → LLMProvider**, with tools, sessions, and hooks plugging in at well-defined points.

### Core Components

- **`nanobot/agent/loop.py`** — `AgentLoop`: the main orchestrator. Receives inbound messages, manages sessions, builds context, dispatches to `AgentRunner`, and handles streaming, commands (`/` prefixes), and tool progress events.
- **`nanobot/agent/runner.py`** — `AgentRunner`: the iterative tool-use loop. Calls the LLM provider, executes tool calls, handles retries, length recovery, mid-turn injection, and auto-compaction. Works with `AgentRunSpec` configs.
- **`nanobot/agent/hook.py`** — `AgentHook` / `AgentHookContext`: lifecycle hooks (`before_iteration`, `on_stream`, `before_execute_tools`, etc.) that channels and the loop itself use to customize behavior without subclassing.
- **`nanobot/nanobot.py`** — `Nanobot`: programmatic facade (`bot = Nanobot.from_config(); result = await bot.run("...")`).

### Providers (`nanobot/providers/`)

LLM providers all implement `LLMProvider` (in `base.py`). Key methods: `complete()` and `stream()`.

- `factory.py` dispatches to the right backend based on provider registry config.
- Provider backends: `openai_compat` (default for most), `anthropic`, `azure_openai`, `github_copilot`, `openai_codex`.
- `registry.py` defines provider specs (name, backend type, OAuth/local flags).
- `transcription.py` handles Whisper-based audio transcription.

### Channels (`nanobot/channels/`)

Chat platform integrations (Telegram, Discord, Slack, Feishu, WeChat, QQ, DingTalk, WhatsApp, Matrix, MSTeams, Email, WebSocket). Each implements `BaseChannel` and communicates via the `MessageBus`.

- `registry.py` auto-discovers built-in channels via `pkgutil` and external plugins via `entry_points`.
- `manager.py` manages channel lifecycle (start/stop all configured channels).

### Tools (`nanobot/agent/tools/`)

Agent tools implement the `Tool` base class (in `base.py`) and register via `ToolRegistry`.

Built-in tools: `read_file`, `write_file`, `edit_file`, `exec` (shell), `grep`, `glob`, `list_dir`, `web_search`, `web_fetch`, `ask` (user interaction), `spawn` (subagents), `notebook`, `cron`, `message`, `self` (agent self-reflection), `mcp` (Model Context Protocol), `file_state` (file tracking), `sandbox`.

### Message Bus (`nanobot/bus/`)

Event-driven communication between channels and the agent loop. `InboundMessage` and `OutboundMessage` are the core event types. `MessageBus` is a simple async queue.

### Sessions (`nanobot/session/`)

`SessionManager` manages per-chat conversation history keyed by `{channel}:{chat_id}`. Sessions are persisted as JSON files.

### Other Key Modules

- **`nanobot/config/`** — Config loading (`loader.py`), schema (`schema.py` — Pydantic models), paths. Config lives at `~/.nanobot/config.json`.
- **`nanobot/agent/context.py`** — `ContextBuilder`: assembles system prompt, skills, workspace context, and session history into messages for the LLM.
- **`nanobot/agent/memory.py`** — `Dream` and `Consolidator`: two-stage memory system (dream = long-term memory consolidation, consolidator = short-term).
- **`nanobot/agent/autocompact.py`** — `AutoCompact`: automatically compresses session history when context grows too large.
- **`nanobot/agent/subagent.py`** — `SubagentManager`: spawns child agent loops for delegated tasks.
- **`nanobot/agent/skills.py`** — Skill discovery and loading from `nanobot/skills/` directory.
- **`nanobot/cron/`** — Scheduled task service (`CronService`).
- **`nanobot/heartbeat/`** — Keeps long-running agent turns alive with periodic heartbeats.
- **`nanobot/security/`** — Network security, path validation, sandboxing.
- **`nanobot/cli/`** — CLI commands via Typer (`commands.py`, `onboard.py` for setup wizard).
- **`nanobot/api/`** — OpenAI-compatible HTTP API server (`aiohttp`).
- **`nanobot/command/`** — Command routing for `/` prefixed user commands.

### Skills (`nanobot/skills/`)

Markdown-based skill definitions (cron, github, memory, summarize, weather, tmux, etc.) plus a `skill-creator` for generating new ones. Skills are loaded by `agent/skills.py` and injected into the system prompt.

## Code Style

- Python 3.11+, line length 100 chars
- Ruff linting with rules E, F, I, N, W (E501 ignored)
- Asyncio throughout; pytest with `asyncio_mode = "auto"`
- Prefer simple, readable code over clever abstractions

## Branching

- `main` — stable releases; target for bug fixes and docs
- `nightly` — experimental features; target for new features and refactors
- When in doubt, target `nightly`
