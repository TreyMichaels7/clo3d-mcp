# AGENTS.md

## What this is
MCP server (`src/clo3d_mcp/`) + single-file CLO3D plugin (`plugin/clo3d_mcp_plugin.py`) bridged by file-based IPC. No sockets, no CI, no lint/typecheck config.

Agent onboarding (any harness): `docs/agent-onboarding.md`. Garment build/iterate workflow: `.claude/skills/clo-render-iterate/SKILL.md` (also loaded by OpenCode).

## Commands
- `uv sync --dev` — install dev deps
- `uv run pytest tests/ -v` — full suite (no CLO3D needed, uses `MockCLO3DServer` in `tests/conftest.py`)
- `uv run clo3d-mcp` — run server locally (entry: `clo3d_mcp:main` → `server.py:mcp.run()`)

## Architecture (file IPC)
- Comm dir: `%TEMP%/clo3d_mcp` on Windows, `/tmp/clo3d_mcp` on macOS/Linux (override: `CLO3D_MCP_DIR`). WSL auto-detects Windows temp in `connection.py:_find_comm_dir`.
- Plugin `serve()` blocks CLO3D's main thread (CLO's embedded Python doesn't run background threads while idle). It exits after 1h idle or on the `stop` command (`stop_server` tool). User must re-run the script in CLO3D for each batch.
- Protocol: server writes `request.json` (`{id, type, params}`), plugin polls and writes `response.json` (`{id, status, result}`). Both use atomic write (`.tmp` + rename). Response `id` must match request; stale ids are skipped.
- Timeout 180s (`TIMEOUT` in `connection.py`) — sim/export can take minutes. Retries: 3x with 1s delay; `ping()` uses 1 retry and returns bool instead of raising.

## Gotchas
- **Plugin must stay a single file with stdlib only.** It runs in CLO3D's built-in Python (Script > Script Editor > Run). Never add imports beyond stdlib + CLO3D's `export_api`, `fabric_api`, `import_api`, `pattern_api`, `utility_api`.
- **Tool names ≠ command names.** `server.py` remaps several: `save_project`→`save_file`, `get_pattern_bounding_box`→`get_bounding_box`, `assign_fabric_to_pattern`→`assign_fabric`. When adding/renaming a tool, update both sides.
- **`CLO3DConnection` is a singleton** (`__new__` guards `_initialized`). Tests bypass it via `CLO3DConnection.__new__` + manual attrs (see `test_connection.py:_make_conn`) because `get_connection()` caches the global. Follow that pattern; don't construct normally in tests.
- **Mock covers only 7 commands** (`ping`, `get_project_info`, `get_pattern_count/list`, `get_colorways`, `simulate`, `export_obj`). Adding a server tool with a new command type requires adding a handler in `conftest.py:MockCLO3DServer._handlers`.
- **`sdk/` is third-party CLO Virtual Fashion code, NOT MIT.** Don't relicense or claim rights; see `NOTICE`. Never edit there unless building against the plugin interface.
