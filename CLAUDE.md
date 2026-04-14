# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Google Workspace MCP Server — a production-grade FastMCP 3.x server exposing 12 Google Workspace services (Gmail, Calendar, Drive, Docs, Sheets, Slides, Chat, Tasks, Forms, Contacts, Apps Script, Custom Search) as 111 MCP tools across 3 tiers (core/extended/complete). Built on Python 3.10+, async-first, with OAuth 2.0/2.1 multi-user support.

## Commands

```bash
# Install
uv sync                          # production deps
uv sync --group dev              # dev deps (pytest, ruff)

# Run server
uv run main.py                              # stdio mode (Claude Desktop)
uv run main.py --transport streamable-http  # HTTP mode on :8000
uv run main.py --tool-tier core             # limit to core tools
uv run main.py --tools gmail drive          # specific services only

# Test
uv run pytest                                          # all tests
uv run pytest tests/gmail/                             # one service
uv run pytest tests/gmail/test_draft_gmail_message.py  # one file
uv run pytest -k "test_name"                           # by name pattern

# Lint & format
uv run ruff check .          # lint
uv run ruff check --fix      # auto-fix
uv run ruff format           # format
```

## Architecture

**Entry points:**
- `main.py` — Local/stdio server (CLI arg parsing, tool tier selection)
- `fastmcp_server.py` — FastMCP Cloud deployment (enforces OAuth 2.1 + stateless)

**Core framework (`core/`):**
- `server.py` — `SecureFastMCP` instance with middleware stack
- `tool_registry.py` — Tool registration + filtering by tier/service
- `tool_tiers.yaml` — YAML config mapping tool names to core/extended/complete tiers
- `utils.py` — `handle_http_errors()` decorator, `UserInputError`, shared types

**Auth (`auth/`):**
- `service_decorator.py` — `@require_google_service()` and `@require_multiple_services()` decorators that inject authenticated Google API clients
- `google_auth.py` — OAuth flow, token refresh, service building
- `scopes.py` — OAuth scope definitions and scope group management
- `oauth21_session_store.py` — Multi-user session/token storage (memory/disk/valkey)

**Service modules** (`gmail/`, `gdrive/`, `gcalendar/`, `gdocs/`, `gsheets/`, `gslides/`, `gforms/`, `gchat/`, `gtasks/`, `gcontacts/`, `gappsscript/`, `gsearch/`):
Each contains `*_tools.py` with `@server.tool()` definitions and optional `*_helpers.py` for shared logic.

## Key Patterns

**Tool definition:** Every tool uses `@server.tool()` + `@require_google_service("service_name", "scope_group")`. The decorator injects an authenticated `service` parameter. Tools return JSON strings. Parameters use `pydantic.Field` for descriptions.

**Multi-service tools:** Use `@require_multiple_services([...])` with `param_name` to inject multiple Google API clients.

**Error handling:** `@handle_http_errors("tool_name")` wraps Google API calls, catching 400/403/404/429/500+ and surfacing them as `ToolExecutionError`. Use `UserInputError` for client-friendly validation errors.

**Async pattern:** Blocking Google API calls run via `await asyncio.get_event_loop().run_in_executor(None, lambda: ...)`.

**Tool tiers:** Defined in `core/tool_tiers.yaml`. Core (~30 tools) for essentials, extended (~50) for management/batch ops, complete (111) for full API access.

## Code Style

- PEP 8 + ruff (120 col line length)
- Mandatory type hints
- f-strings only (no `%` formatting)
- Async-first; use `run_in_executor` for blocking Google API calls
- Tool names: imperative, concise
- No secrets/PII in logs or exceptions; least-privilege OAuth scopes

## Testing

- Framework: pytest + pytest-asyncio
- Tests mirror service structure under `tests/`
- Mock Google APIs — no live API calls in tests
- Use `monkeypatch` for env vars, `Mock()` for services, `tmp_path` for file fixtures

## Deployment

- **Server URL:** `https://mcp.musicalmycology.org/mcp`
- **OAuth Client ID:** `609840661946-avn9dpgdi9dj941e7bnrlvrdigaqunaq.apps.googleusercontent.com`
- Deployed via Docker Compose with Cloudflare tunnel (cloudflared sidecar)
- Google Cloud Console for OAuth config: APIs & Credentials → OAuth 2.0 Client IDs

## Pending: OAuth Redirect URI

OAuth redirect URI updated in docker-compose.yml to `https://mcp.musicalmycology.org/oauth2callback` (commit d6f96b6) so remote clients can complete Google auth instead of being sent to localhost.

**Still needed:** Add `https://mcp.musicalmycology.org/oauth2callback` to the authorized redirect URIs in Google Cloud Console for OAuth client `609840661946-avn9dpgdi9dj941e7bnrlvrdigaqunaq`. Without this, Google will reject the callback.

After updating Google Console, git pull on the server and `docker-compose down && docker-compose up -d` to pick up the new env var.
