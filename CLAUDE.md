# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run all integration tests (mocked Garmin API)
uv run pytest tests/integration/

# Run a single test module
uv run pytest tests/integration/test_workouts_tools.py -v

# Run a single test function
uv run pytest tests/integration/test_workouts_tools.py::test_get_workouts_tool -v

# Run unit tests
uv run pytest tests/unit/ -v

# Run e2e tests (requires real Garmin credentials)
uv run pytest tests/e2e/ -m e2e -v

# Run the MCP server locally
uv run garmin-mcp

# Authenticate interactively (one-time setup or token refresh)
uv run garmin-mcp-auth

# Verify saved tokens
uv run garmin-mcp-auth --verify

# Inspect/test tools via MCP Inspector
npx @modelcontextprotocol/inspector uv run garmin-mcp
```

## Architecture

The server is a [FastMCP](https://github.com/jlowin/fastmcp) application that wraps the [python-garminconnect](https://github.com/cyberjunky/python-garminconnect) library and exposes Garmin Connect data as MCP tools.

### Module pattern

Each domain area is a standalone module in `src/garmin_mcp/` with two required functions:

- `configure(client)` — injects the shared `Garmin` client instance into a module-level global
- `register_tools(app) -> app` — registers `@app.tool()` decorated async functions and returns the app

`src/garmin_mcp/__init__.py` (`main()`) orchestrates startup: authenticates, calls `configure()` on every module, creates the `FastMCP` app, then calls `register_tools()` on every module. `workout_templates.py` is the only module that uses `register_resources()` instead (MCP resources, not tools).

### Tool response convention

All tool functions return `str` (JSON-serialized or plain text). When the Garmin API returns large raw objects, tools curate them down to essential fields before serializing. See `_curate_workout_summary()` in `workouts.py` for an example.

### Authentication flow

OAuth tokens are stored at `~/.garminconnect` (configurable via `GARMINTOKENS` env var). On startup, `init_api()` tries token login first; if that fails, it falls back to email/password login. In non-interactive environments (MCP subprocess) without tokens, it exits cleanly with an actionable error rather than hanging.

### Test structure

- `tests/integration/` — one file per module, using FastMCP's `app.call_tool()` with a `Mock` Garmin client. The `conftest.py` / `app_factory` fixture handles boilerplate. Tests are `@pytest.mark.asyncio` (auto mode).
- `tests/unit/` — isolated logic tests (e.g., `auth_cli`, `token_utils`)
- `tests/e2e/` — require real credentials, skipped by default, opt-in with `-m e2e`
- `tests/fixtures/garmin_responses.py` — shared mock API response data

### Adding a new tool

1. Add the async function inside the `register_tools()` body of the relevant module, decorated with `@app.tool()`.
2. Keep the return type `str`. Curate large API responses before returning.
3. Add a corresponding integration test in `tests/integration/test_<module>_tools.py` following the existing pattern.
