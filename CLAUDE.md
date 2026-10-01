# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Private MCP server for Resolume Arena / Avenue control over **REST, WebSocket, and OSC**.
Forked to `jalsaffar/resolume-mcp` (origin) from `drohi-r/resolume-mcp` (upstream).

See `../CLAUDE.md` for the shared MCP conventions and `AGENTS.md` for the short rule list.

## Commands

```bash
uv sync
uv run python -m pytest -v
uv run python -m pytest tests/test_advanced_output_xml.py -v   # single file
uv run python -m resolume_mcp.server
```

`pytest-asyncio` is a dev dependency — async tests are expected here.

## Architecture

- **`config.py`** — `ResolumeConfig` frozen dataclass + `load_config()`. Derives `http_base_url`
  and `websocket_url` from one host/port pair, so HTTPS/WSS flip together via `RESOLUME_USE_HTTPS`.
- **`client.py`** (173 lines) — `ResolumeClient` (httpx) plus OSC encoders. `normalize_api_path`
  and `join_url` exist so tools can pass either `/composition/...` or `composition/...`; route new
  paths through them rather than string-concatenating a URL.
- **`server.py`** (~3300 lines) — the tool surface. Large because Resolume's object model is large
  (composition → groups → layers → clips, plus decks, columns, effects, and Advanced Output).
- **`advanced_output_xml.py`** (539 lines) — see below.

### Three transports, three jobs

REST is request/response and is the default. WebSocket carries subscriptions (`subscribe_*` /
`unsubscribe_*` tool families) for live parameter state. OSC is fire-and-forget. Picking the wrong
one produces a tool that appears to work but never reflects live changes — match the transport to
whether the caller needs confirmation or a stream.

### Advanced Output edits files, not the API

`advanced_output_xml.py` reads and writes Resolume's **preference XML on disk**
(`AdvancedOutput.xml`, `slices.xml` under `~/Documents/Resolume Arena/Preferences/`), because
screen/slice geometry is not fully exposed over the REST API. This is a fundamentally different
risk profile from the rest of the server: it mutates files Resolume owns, and Resolume may hold
its own in-memory copy. Backup/diff/preview/restore tools exist for exactly this reason — prefer
them over a bare write, and keep them working.

## Config

| Env var | Default |
|---|---|
| `RESOLUME_HOST` | `127.0.0.1` |
| `RESOLUME_HTTP_PORT` | `8080` |
| `RESOLUME_OSC_PORT` | `7000` |
| `RESOLUME_ALLOWED_HOSTS` | `127.0.0.1,localhost,::1` |
| `RESOLUME_USE_HTTPS` | `0` |
| `RESOLUME_DOCUMENTS_ROOT` | `~/Documents/Resolume Arena` |
| `RESOLUME_ADVANCED_OUTPUT_XML` | `<documents>/Preferences/AdvancedOutput.xml` |
| `RESOLUME_SLICES_XML` | `<documents>/Preferences/slices.xml` |

## Domain facts worth knowing

- Hierarchy: composition → groups → layers → clips. Only **one clip plays per layer** — triggering
  a new clip ejects the current one. Columns are alternatives at the composition/group level.
- New layers default to video/opacity **0.5**, which reads as "transparent/faded" when someone
  reports a bug.
- Default blend mode is Add.
- A `#` in an entity name auto-displays its position in the Resolume UI; the API stores the
  literal `#`.

## Docs and skills

`docs/` holds `GETTING-STARTED.md`, `LIVE-VALIDATION.md` (what is verified against real hardware),
and `SESSION-HANDOVER.md`. `.claude/skills/` provides seven operator skills — advanced output
setup, deck control, festival recovery, output routing, warp alignment, playback prep/busking, and
show recovery/triage — which load automatically when working in this directory.
