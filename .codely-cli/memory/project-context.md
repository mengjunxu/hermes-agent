# Hermes Agent — Project Context

## Project Overview

**Hermes Agent** is a self-improving personal AI agent built by [Nous Research](https://nousresearch.com). It runs a single agent core across a CLI, a messaging gateway (Telegram, Discord, Slack, WhatsApp, Signal, Matrix, and ~20 other platforms), a TUI (Ink/React), an Electron desktop app, and a web dashboard. The agent learns across sessions (memory + skills), delegates to subagents, runs scheduled jobs (cron), and drives a real terminal and browser. It is extended primarily through **plugins and skills**, not by growing the core.

- **Version:** 0.19.0 (pyproject.toml)
- **License:** MIT
- **Languages:** Python 3.11–3.13 (primary), TypeScript/React (TUI, desktop, web)
- **Package manager:** uv (Python), npm workspaces (JS/TS)
- **Build system:** setuptools (Python), tsc/vite (JS/TS)
- **Repo:** [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- **Docs:** [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com)
- **Key docs in repo:** `AGENTS.md` (development guide for AI assistants), `CONTRIBUTING.md` (contributor guide)

### Two Sacred Design Principles

1. **Per-conversation prompt caching is sacred.** Never mutate past context, swap toolsets, or rebuild the system prompt mid-conversation — it invalidates the cached prefix and multiplies cost. The only exception is context compression.
2. **The core is a narrow waist; capability lives at the edges.** Every model tool ships on every API call, so the bar for new core tools is high. New capability should arrive as a CLI command + skill, a service-gated tool, a plugin, or an MCP server — not as core surface.

## Building and Running

### Install (development)

```bash
# Standard installer (creates managed venv + clones repo)
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
cd "${HERMES_HOME:-$HOME/.hermes}/hermes-agent"

# Add dev/test extras
uv pip install -e ".[all,dev]"

# Optional: JS/TS dependencies (browser tools, TUI, desktop, web)
npm install
```

### Configure

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env
echo "OPENROUTER_API_KEY=***" >> ~/.hermes/.env
```

### Run

```bash
hermes doctor          # diagnostics
hermes chat -q "Hello" # quick chat
hermes                 # interactive CLI
hermes --tui           # Ink-based TUI
hermes dashboard       # web dashboard (localhost)
```

### Tests

```bash
# ALWAYS use the wrapper — never call pytest directly
scripts/run_tests.sh                              # full suite (CI parity)
scripts/run_tests.sh tests/gateway/               # one directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # one test
scripts/run_tests.sh -v --tb=long                 # pass-through pytest flags
```

The wrapper enforces: unset credential vars, TZ=UTC, LANG=C.UTF-8, `-n auto` xdist workers, subprocess-per-test-file isolation. Direct `pytest` diverges from CI and has caused "works locally, fails in CI" incidents.

### JS/TS (TUI, desktop, web)

```bash
cd ui-tui
npm install && npm run dev        # TUI watch mode
npm run typecheck                 # tsc --noEmit
npm run lint                      # eslint
npm test                          # vitest
```

### Linting / Type-checking

```bash
# Python
ruff check .                      # only PLW1514 (unspecified-encoding) is enforced
ty check                          # type checker (ty == astral's type checker)

# JS/TS (per workspace)
npm run --ws check                # typecheck + lint across all workspaces
npm run --ws fix                  # auto-fix across all workspaces
```

## Key Architecture

### Core Files (load-bearing entry points)

| File | Purpose |
|------|---------|
| `run_agent.py` | `AIAgent` class — core conversation loop (~12k LOC). Entry: `run_conversation()` |
| `cli.py` | `HermesCLI` class — interactive CLI orchestrator (~11k LOC). Uses Rich + prompt_toolkit |
| `model_tools.py` | Tool orchestration, `discover_builtin_tools()`, `handle_function_call()` |
| `toolsets.py` | Toolset definitions, `TOOLSETS` dict, `_HERMES_CORE_TOOLS` list |
| `hermes_state.py` | `SessionDB` — SQLite session store with FTS5 full-text search |
| `hermes_constants.py` | `get_hermes_home()`, `display_hermes_home()` — profile-aware paths |
| `hermes_logging.py` | `setup_logging()` — agent.log / errors.log / gateway.log (profile-aware) |
| `batch_runner.py` | Parallel batch processing for trajectory generation |

### File Dependency Chain

```
tools/registry.py  (no deps — imported by all tool files)
       ↑
tools/*.py  (each calls registry.register() at import time)
       ↑
model_tools.py  (imports tools/registry + triggers tool discovery)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### Directory Structure

```
agent/          # Agent internals (prompt_builder, context_compressor, memory, caching, display)
hermes_cli/     # CLI subcommands, setup wizard, config, plugins loader, skin engine, commands registry
tools/          # Tool implementations — self-registering via tools/registry.py
gateway/        # Messaging gateway — run.py + session.py + platforms/
plugins/        # Plugin system (memory providers, model providers, context engines, etc.)
skills/         # Built-in skills (bundled, active by default)
optional-skills/# Official optional skills (shipped but not active by default)
ui-tui/         # Ink (React) terminal UI — `hermes --tui`
tui_gateway/    # Python JSON-RPC backend for the TUI
apps/desktop/   # Electron desktop app
acp_adapter/    # ACP server (VS Code / Zed / JetBrains integration)
cron/           # Scheduler — jobs.py, scheduler.py
scripts/        # run_tests.sh, install.sh, install.ps1, release.py, auxiliary scripts
tests/          # Pytest suite (~17k tests across ~900 files)
tests-js/       # Vitest suite (JS/TS tests)
web/            # Web dashboard frontend (React)
website/        # Docusaurus docs site
```

### User Config Locations

| Path | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | Settings (model, terminal, toolsets, compression, etc.) |
| `~/.hermes/.env` | API keys and secrets ONLY |
| `~/.hermes/auth.json` | OAuth credentials (Nous Portal) |
| `~/.hermes/skills/` | All active skills (bundled + hub-installed + agent-created) |
| `~/.hermes/memories/` | Persistent memory (MEMORY.md, USER.md) |
| `~/.hermes/state.db` | SQLite session database |
| `~/.hermes/cron/` | Scheduled job data |
| `~/.hermes/logs/` | agent.log (INFO+), errors.log (WARNING+), gateway.log |

### Agent Loop

```
User message → AIAgent.run_conversation()
  ├── Build system prompt (agent/prompt_builder.py)
  ├── Build API kwargs (model, messages, tools, reasoning config)
  ├── Call LLM (OpenAI-compatible API)
  ├── If tool_calls in response:
  │     ├── Execute each tool via registry dispatch (model_tools.py)
  │     ├── Add tool results to conversation
  │     └── Loop back to LLM call
  ├── If text response:
  │     ├── Persist session to SQLite DB
  │     └── Return final_response
  └── Context compression if approaching token limit
```

### Tool System

- **Self-registering:** Each `tools/*.py` calls `registry.register()` at import time. `model_tools.py` triggers discovery.
- **Toolset grouping:** Tools grouped into toolsets (`web`, `terminal`, `file`, `browser`, etc.) — enabled/disabled per platform.
- **Adding a core tool:** Create `tools/your_tool.py` with `registry.register(...)`, then add the tool name to `_HERMES_CORE_TOOLS` (or a new toolset) in `toolsets.py`. Both steps are required.
- **Footprint Ladder (preference order):** Extend existing code → CLI command + skill → service-gated tool (`check_fn`) → plugin → MCP server → new core tool (last resort).
- **Tool handlers MUST return a JSON string.**
- **Agent-level tools** (todo, memory): intercepted by `run_agent.py` before `handle_function_call()`.

### Slash Commands

All slash commands defined in a central `COMMAND_REGISTRY` in `hermes_cli/commands.py` as `CommandDef` objects. Every consumer (CLI dispatch, gateway, Telegram menu, Slack routing, autocomplete, help) derives from this registry automatically. Adding an alias requires only updating the `aliases` tuple — no other file changes.

### Config System

- **`~/.hermes/config.yaml`** — all behavioral settings (model, terminal, toolsets, compression, display, etc.)
- **`~/.hermes/.env`** — secrets ONLY (API keys, tokens, passwords). Never put behavioral config here.
- **Three config loaders:** `load_cli_config()` (CLI mode, in `cli.py`), `load_config()` (subcommands/setup, in `hermes_cli/config.py`), direct YAML read (gateway runtime, in `gateway/run.py`). Check which loader your code path uses.
- **Profiles:** Multiple isolated instances, each with its own `HERMES_HOME`. Use `get_hermes_home()` for all paths — never hardcode `~/.hermes`.
- **Adding config:** Add to `DEFAULT_CONFIG` in `hermes_cli/config.py`. Bump `_config_version` only if migrating/transforming existing user config.

### Plugin System

- **General plugins** (`plugins/<name>/`): register lifecycle hooks (`pre_tool_call`, `post_tool_call`, `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`), tools, CLI subcommands via `register(ctx)`. Discovered from `~/.hermes/plugins/`, `./.hermes/plugins/`, pip entry points.
- **Memory-provider plugins** (`plugins/memory/<name>/`): implement `MemoryProvider` ABC. **No new in-tree providers accepted** — ship as standalone repos.
- **Model-provider plugins** (`plugins/model-providers/<name>/`): each calls `providers.register_provider(ProviderProfile(...))`. Lazy, separate discovery.
- **Policy:** Plugins MUST NOT modify core files. Expand the generic plugin surface instead. No new third-party-product plugins in-tree.

### Skills

- **`skills/`** — bundled, active by default. Organized by category.
- **`optional-skills/`** — shipped but not active. Installed via `hermes skills install official/<category>/<skill>`.
- **SKILL.md** frontmatter: `name`, `description` (≤60 chars), `version`, `author`, `platforms`, `metadata.hermes.*`.
- **Standards:** Use native Hermes tool names in prose (`` `terminal` ``, `` `read_file` ``, `` `search_files` ``, etc. — NOT `grep`, `cat`, `sed`). Scripts in `scripts/`, references in `references/`. Tests at `tests/skills/test_<skill>_skill.py`.

### Other Subsystems

- **Cron** (`cron/`): `jobs.py` (job store) + `scheduler.py` (tick loop). 3-minute hard interrupt on cron sessions. File lock prevents duplicate ticks.
- **Kanban** (`plugins/kanban/`): Durable SQLite-backed board for multi-agent collaboration. Board is the hard boundary; tenant is soft namespace.
- **Curator** (`agent/curator.py`): Background skill-maintenance. Tracks usage, auto-archives stale skills. Never deletes — archives are restorable.
- **Delegation** (`tools/delegate_tool.py`): Spawns subagents with isolated context. Roles: `leaf` (default) and `orchestrator`.
- **Skin/Theme** (`hermes_cli/skin_engine.py`): Data-driven CLI visual customization. Pure data — no code changes needed to add a skin.

## Development Conventions

### Python Style

- PEP 8 (practical exceptions, no strict line length enforcement)
- Comments only for non-obvious intent — never narrate what code does
- Catch specific exceptions; log with `logger.warning()`/`logger.error()`, use `exc_info=True` for unexpected errors
- All paths must use `get_hermes_home()` / `display_hermes_home()` — never hardcode `~/.hermes`
- `ruff` enforces only `PLW1514` (must specify `encoding=` on `open()`/`read_text()`/`write_text()`)

### TypeScript Style

- Prefer nanostores over component state for shared/reused state
- Each feature owns its atoms; shared state in `src/store`
- Prefer interfaces for public props; extend React primitives (`React.ComponentProps<'button'>`)
- Table-driven beats condition ladders
- `src/app` owns routes/pages; `src/store` owns shared atoms; `src/lib` owns pure helpers

### Commit Messages

Conventional Commits: `<type>(<scope>): <description>`

Types: `fix`, `feat`, `docs`, `test`, `refactor`, `chore`
Scopes: `cli`, `gateway`, `tools`, `skills`, `agent`, `install`, `security`, etc.

### Branch naming

```
fix/description    # Bug fixes
feat/description   # New features
docs/description   # Documentation
test/description   # Tests
refactor/description
```

### Dependency Pinning

All dependencies are **exact-pinned** (`==X.Y.Z`) for core deps, or `>=floor,<next_major` for ranges. No bare `>=X.Y.Z` without a ceiling. Git URLs pinned to commit SHA. GitHub Actions pinned to SHA + version comment. Run `uv lock` after changing deps. Policy established after litellm compromise (March 2026) and Mini Shai-Hulud worm (May 2026).

### Testing Rules

- **ALWAYS use `scripts/run_tests.sh`** — never `pytest` directly
- Tests must not write to `~/.hermes/` — the `_isolate_hermes_home` autouse fixture redirects to temp dir
- Tests that read JS/TS artifacts (`package.json`, `.ts`/`.tsx`) belong in `tests-js/` (vitest), not `tests/*.py`
- **No change-detector tests** — assert invariants/behavior, not snapshot values (model lists, config versions, counts)
- **No reading source code in tests** — test behavior, not source text shape
- E2E validation over mocks for resolution chains, config propagation, security boundaries
- Profile tests must also mock `Path.home()` + set `HERMES_HOME`
- Flake policy: auto-retries failing test FILE once; pass-on-retry printed as `⚠ FLAKY`

### Cross-Platform

Hermes runs on Linux, macOS, native Windows, WSL2, and Termux. Critical rules:

- **Never use `os.kill(pid, 0)`** for liveness — on Windows it sends Ctrl+C to the entire console process group. Use `psutil.pid_exists(pid)`.
- Use `shutil.which()` before shelling out — don't assume Windows has POSIX tools
- Guard `termios`/`fcntl` with `try/except (ImportError, NotImplementedError)`
- Use `pathlib.Path` instead of string concatenation with `/`
- Use `sys.executable` to invoke Python, never shebangs
- `termios`, `fcntl`, `os.setsid`, `os.killpg`, `os.fork`, `os.getuid` are Unix-only
- Signals `SIGALRM`, `SIGCHLD`, `SIGHUP`, `SIGUSR1`, `SIGUSR2`, `SIGPIPE`, `SIGQUIT`, `SIGKILL` don't exist on Windows
- Run `scripts/check-windows-footguns.py` before PRs
- Keep `scripts/install.sh` and `scripts/install.ps1` in lockstep

### Key Pitfalls

- **Don't hardcode `~/.hermes`** — breaks profiles. Use `get_hermes_home()`.
- **Don't use `simple_term_menu`** for new code — use `hermes_cli/curses_ui.py`.
- **Don't use `\033[K`** (ANSI erase-to-EOL) — leaks under prompt_toolkit. Use space-padding.
- **Don't hardcode cross-tool references** in schema descriptions — add dynamically in `get_tool_definitions()`.
- **Don't wire in dead code** without E2E validation against real imports.
- **Squash merges from stale branches** silently revert fixes — ensure branch is up to date with `main`.
- **`_last_resolved_tool_names`** is a process-global in `model_tools.py` — may be temporarily stale during subagent runs.
- **Gateway has TWO message guards** — both must bypass approval/control commands.
