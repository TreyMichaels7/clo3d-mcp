# Onboarding an agent to the CLO3D loop

How to get any MCP-capable agent (Claude Code, OpenCode/Muse, Cursor, a custom stdio client) driving CLO3D through this server. The garment-specific know-how lives in `.claude/skills/clo-render-iterate/SKILL.md`; this doc is the plumbing.

## 1. Prereqs (human, once per batch)

1. Open CLO3D and load an avatar through the UI (File > Open > Avatar). Don't import avatars via the API — it has crashed CLO.
2. Script > Script Editor > open `plugin/clo3d_mcp_plugin.py` > Run.
3. Console must show `Serving commands in: /tmp/clo3d_mcp` (Windows: `%TEMP%\clo3d_mcp`).
4. CLO's UI is now frozen. That's expected — `serve()` blocks the main thread because CLO's embedded Python doesn't run background threads while idle.

## 2. Connect the agent

The server is a stdio MCP server started with `uv run clo3d-mcp` from the repo root.

**Claude Code**
```bash
claude mcp add clo3d -- uv --directory /path/to/clo3d-mcp run clo3d-mcp
```

**OpenCode** — `opencode.json` is checked in at the repo root. Run `opencode` from the repo root and the `clo3d` server starts automatically. Skills under `.claude/skills/` and `AGENTS.md` are also picked up natively.

**Any other stdio client** — command `uv`, args `["--directory", "/path/to/clo3d-mcp", "run", "clo3d-mcp"]`. Set env `CLO3D_MCP_DIR` only if you need to override the comm dir (e.g. WSL pointing at Windows temp; `connection.py:_find_comm_dir` auto-detects the common case).

**Tool naming.** Tools are documented by bare name (`get_project_info`). Harnesses prefix them: `mcp__clo3d__get_project_info` (Claude Code), `clo3d_get_project_info` (OpenCode). Same tool.

## 3. Smoke test

Call `get_project_info`. Expect JSON within ~10s.

- Hangs → plugin isn't polling. Check `/tmp/clo3d_mcp/request.json` exists and gets consumed. Re-run the plugin script in CLO.
- `ping` returns a bool instead of raising; use it for "is CLO alive" checks.

## 4. Load the skill

- OpenCode / Claude Code: `.claude/skills/clo-render-iterate/SKILL.md` is discovered automatically. Invoke it before touching patterns.
- Harness without skills: paste SKILL.md into the system prompt or the harness's rules file. It's ~60 lines; it fits.

## 5. Session etiquette

- Save the `.zprj` (`save_project`) after every pass. CLO has crashed mid-session.
- Call `stop_server` at the end of a batch. After that the human must re-run the plugin script for the next batch.
- Plugin auto-exits after 1h idle.
- Per-call timeout is 180s (`connection.py:TIMEOUT`). Sims and exports can approach it — keep `simulate` steps ≤600 per call and split long work.
- Never call anything that opens a CLO dialog (`ExportSnapshot3D`, file pickers). The UI is frozen; the dialog never returns and the call times out.

## 6. Model requirements

- **Vision input.** The loop's only feedback is the rendered PNG (`ExportThumbnail3D` via `call_api`). A model that can't view images can build and simulate but can't iterate — it must hand the render path to the human.
- **Tool-call budget.** One fix→sim→render pass is 10–20 calls; a session is up to ~8 passes. Harness must allow long agentic runs.
- **`call_api` for anything unwrapped.** The 33 MCP tools cover common ops; `call_api(module, function, args)` reaches the rest of `export_api`, `fabric_api`, `import_api`, `pattern_api`, `utility_api`. Signatures are in `sdk/` (third-party CLO code — read-only, see `NOTICE`).

## 7. Extending the server

Adding a tool touches three places (see `AGENTS.md`):
1. `src/clo3d_mcp/server.py` — the tool (note tool name ≠ command name for some; keep the remap table in sync).
2. `plugin/clo3d_mcp_plugin.py` — the command handler. Stdlib + CLO APIs only; single file.
3. `tests/conftest.py:MockCLO3DServer._handlers` — mock handler so `uv run pytest tests/ -v` covers it without CLO.
