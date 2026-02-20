# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.2] - 2026-02-20

### Added

- Streamable HTTP transport mode — set `HTTP_PORT` to serve MCP over HTTP with SSE streaming and session management
- Origin validation middleware (MCP 2025-11-25 §4.3) — rejects cross-origin requests with HTTP 403
- `HTTP_HOST` env var for configurable bind address (default: `127.0.0.1`)
- `ALLOWED_ORIGINS` env var for cross-origin allowlist
- 13 new integration tests covering HTTP transport, Origin validation, and session ordering

## [0.1.1] - 2026-02-18

### Added

- Initial release of `@temporal-cortex/cortex-mcp`
- 6 MCP tools: `list_events`, `find_free_slots`, `book_slot`, `expand_rrule`, `check_availability`, `get_availability`
- Lite Mode: zero-infrastructure single Google Calendar setup
- Atomic booking with lock-verify-write conflict prevention via Two-Phase Commit
- TOON-compressed output (~40% fewer tokens than JSON)
- OAuth PKCE flow with local credential persistence
- Deterministic RRULE expansion via Truth Engine (DST-aware, BYSETPOS, leap years)
- RRULE Challenge CLI command for demonstrating edge case handling

[Unreleased]: https://github.com/billylui/temporal-cortex-mcp/compare/mcp-v0.1.2...HEAD
[0.1.2]: https://github.com/billylui/temporal-cortex-mcp/compare/mcp-v0.1.1...mcp-v0.1.2
[0.1.1]: https://github.com/billylui/temporal-cortex-mcp/releases/tag/mcp-v0.1.1
