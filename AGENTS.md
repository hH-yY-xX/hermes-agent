# Hermes Agent - Development Guide

Instructions for AI coding assistants working on the hermes-agent codebase.

## Dev Environment

```bash
source .venv/bin/activate   # or: source venv/bin/activate
```

**Testing:** ALWAYS use `scripts/run_tests.sh` — never call `pytest` directly. It enforces hermetic env parity with CI (unset API keys, TZ=UTC, LANG=C.UTF-8, 4 xdist workers).

**Config:** `~/.hermes/config.yaml` (settings), `~/.hermes/.env` (API keys only).
**Logs:** `~/.hermes/logs/agent.log` (INFO+), `errors.log` (WARNING+), `gateway.log`.

## Project Structure (key files)

```
run_agent.py          # AIAgent class — core conversation loop (~12k LOC)
model_tools.py        # Tool orchestration, handle_function_call()
toolsets.py           # Toolset definitions, _HERMES_CORE_TOOLS list
cli.py                # HermesCLI class — interactive CLI (~11k LOC)
hermes_state.py       # SessionDB — SQLite session store (FTS5)
hermes_constants.py   # get_hermes_home(), display_hermes_home() — profile-aware paths
hermes_logging.py     # setup_logging() — profile-aware log paths
agent/                # Provider adapters, memory, caching, compression
hermes_cli/           # CLI subcommands, setup wizard, plugins, skin engine
tools/                # Tool implementations — auto-discovered via tools/registry.py
gateway/              # Messaging gateway + platforms/ (telegram, discord, slack, ...)
plugins/              # Plugin system (memory, model-providers, kanban, etc.)
ui-tui/               # Ink (React) terminal UI — `hermes --tui`
tui_gateway/          # Python JSON-RPC backend for TUI
```

**Dependency chain:** `tools/registry.py` → `tools/*.py` → `model_tools.py` → `run_agent.py, cli.py`

## AIAgent (run_agent.py)

Core loop in `run_conversation()` — synchronous, with interrupt checks and budget tracking:
1. LLM call with tool schemas
2. If tool_calls: execute each via `handle_function_call()`, append results
3. If no tool_calls: return content
4. Repeat until `max_iterations` or budget exhausted

Messages: OpenAI format `{"role": "system/user/assistant/tool", ...}`. Reasoning stored in `assistant_msg["reasoning"]`.

## CLI Architecture (cli.py)

- **Rich** for banners, **prompt_toolkit** for input with autocomplete
- Slash commands: central `COMMAND_REGISTRY` in `hermes_cli/commands.py` — all consumers (CLI, gateway, Telegram, Slack, autocomplete) derive from it
- Adding a command: add `CommandDef` to registry → add handler in `cli.py` → add handler in `gateway/run.py` (if gateway-supported)
- Skill slash commands inject as **user message** (not system prompt) to preserve caching

### Adding an alias
Only add to the `aliases` tuple on the existing `CommandDef` — everything updates automatically.

## TUI (ui-tui + tui_gateway)

Activated via `hermes --tui` or `HERMES_TUI=1`. TypeScript (Ink/React) owns screen, Python owns sessions/tools via stdio JSON-RPC.

**Dashboard embeds real `hermes --tui`** via PTY bridge (`hermes_cli/pty_bridge.py`). Do NOT reimplement chat in React — extend Ink instead.

## Adding Tools

**Plugin route (preferred):** `~/.hermes/plugins/<name>/plugin.yaml` + `__init__.py`, use `ctx.register_tool(...)`.

**Core tools (2 files):**
1. `tools/your_tool.py` — `registry.register(name, toolset, schema, handler, check_fn)`
2. `toolsets.py` — add to `_HERMES_CORE_TOOLS` or new toolset (required step!)

All handlers MUST return JSON string. Use `get_hermes_home()` for state paths, `display_hermes_home()` for user-facing paths.

## Configuration

- **config.yaml:** Add to `DEFAULT_CONFIG` in `hermes_cli/config.py`. Bump `_config_version` only for active migrations.
- **.env:** Secrets only — add to `OPTIONAL_ENV_VARS` with metadata.
- **Non-secret settings** belong in `config.yaml`, not `.env`.
- **Config loaders:** `load_cli_config()` (CLI), `load_config()` (most CLI cmds), direct YAML load (gateway).

## Plugins

**General:** `register(ctx)` can register hooks (`pre_tool_call`, `post_tool_call`, etc.), tools (`ctx.register_tool()`), CLI commands (`ctx.register_cli_command()`). Discovery via `discover_plugins()` (side effect of importing `model_tools.py`).

**Memory providers:** `plugins/memory/<name>/` — implement `MemoryProvider` ABC. CLI commands only exposed for active provider.

**Model providers:** `plugins/model-providers/<name>/` — call `providers.register_provider(ProviderProfile(...))`. Lazy discovery on first use.

**Rule:** Plugins MUST NOT modify core files (`run_agent.py`, `cli.py`, `gateway/run.py`, etc.).

## Skills

- `skills/` — built-in, loadable by default
- `optional-skills/` — heavier/niche, installed explicitly via `hermes skills install`

## Toolsets

All in `toolsets.py` as `TOOLSETS` dict. Keys: `browser`, `clarify`, `code_execution`, `cronjob`, `debugging`, `delegation`, `discord`, `feishu_doc`, `file`, `homeassistant`, `image_gen`, `kanban`, `memory`, `messaging`, `moa`, `rl`, `safe`, `search`, `session_search`, `skills`, `spotify`, `terminal`, `todo`, `tts`, `video`, `vision`, `web`, `yuanbao`.

## Delegation

`delegate_task` spawns subagent with isolated context. Synchronous — parent waits for child summary.
- `role="leaf"` (default): no `delegate_task`, `clarify`, `memory`, `send_message`, `execute_code`
- `role="orchestrator"`: can spawn workers (gated by `delegation.orchestrator_enabled`, max depth 2)
- Not durable — use `cronjob` or `terminal(background=True)` for long-running work.

## Curator

Background skill-maintenance: auto-archives stale agent-created skills. Only touches `created_by: "agent"` skills. Pinned skills exempt. Config under `curator:` in config.yaml.

## Cron

Scheduled jobs via `cron/` — agents use `cronjob` tool, users use `hermes cron <verb>`. 3-min hard interrupt. Catchup window: half period, clamped 120s-2h. Sessions pass `skip_memory=True`.

## Kanban

Multi-agent work queue (`hermes kanban <verb>`). Board = hard boundary, Tenant = soft namespace. Dispatcher runs inside gateway by default. Auto-blocks after ~5 consecutive spawn failures.

## Important Policies

### Prompt Caching Must Not Break
- NEVER alter context/toolsets/mid-conversation
- Slash commands that mutate prompt state: default to deferred invalidation, opt-in `--now` for immediate

### Profile-Safe Code
1. Use `get_hermes_home()` for all HERMES_HOME paths (import from `hermes_constants`)
2. Use `display_hermes_home()` for user-facing messages
3. NEVER hardcode `~/.hermes` or `Path.home() / ".hermes"`
4. Tests mocking `Path.home()` must also set `HERMES_HOME` env var

## Known Pitfalls

- **No `simple_term_menu`** — use `hermes_cli/curses_ui.py`
- **No `\033[K`** in spinner code — leaks as `?[K` under prompt_toolkit
- **No cross-tool references** in schema descriptions — tools may be unavailable
- **Gateway has TWO message guards** — approval/control commands must bypass both
- **Squash merges from stale branches** silently revert fixes — update branch first
- **Tests must not write to `~/.hermes/`** — `_isolate_hermes_home` fixture redirects to temp dir

## Testing

```bash
scripts/run_tests.sh tests/gateway/        # one directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # one test
```

**Don't write change-detector tests** — assert invariants/relationships, not specific values/counts that change with every update.
