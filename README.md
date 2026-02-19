[![npm version](https://img.shields.io/npm/v/@temporal-cortex/cortex-mcp)](https://www.npmjs.com/package/@temporal-cortex/cortex-mcp)
[![npm downloads](https://img.shields.io/npm/dm/@temporal-cortex/cortex-mcp)](https://www.npmjs.com/package/@temporal-cortex/cortex-mcp)
[![Smithery](https://smithery.ai/badge/@temporal-cortex/cortex-mcp)](https://smithery.ai/server/@temporal-cortex/cortex-mcp)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

# temporal-cortex-mcp

**The only Calendar MCP server with atomic booking and conflict prevention.**

## The Problem

AI agents double-book your calendar. They hallucinate availability. They silently fail on recurring events that cross DST boundaries, skip leap years, or use `BYSETPOS`. Every other Calendar MCP server is a thin CRUD wrapper that passes these failures through to Google Calendar — no verification, no conflict detection, no safety net.

LLMs get date and time tasks wrong roughly 60% of the time ([AuthenHallu benchmark](https://arxiv.org/abs/2407.12282)). When your AI assistant says "You're free at 2pm" and books a meeting there, it might be wrong — and there's nothing between the LLM's hallucination and your calendar.

## What's Different

- **Atomic booking** — Lock the time slot, verify no conflicts exist, then write. Two agents booking the same 2pm slot? Exactly one succeeds. The other gets a clear error. No double-bookings.
- **Computed availability** — Merges free/busy data across multiple calendars into a single unified view. The AI sees actual availability, not a raw dump of events to misinterpret.
- **Deterministic RRULE expansion** — Handles DST transitions, `BYSETPOS=-1` (last weekday of month), `EXDATE` with timezones, leap year recurrences, and `INTERVAL>1` with `BYDAY`. Powered by [Truth Engine](https://github.com/billylui/temporal-cortex-core), not LLM inference.
- **Token-efficient output** — TOON format compresses calendar data to ~40% fewer tokens than standard JSON, reducing costs and context window usage.

## Quick Start

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "temporal-cortex": {
      "command": "npx",
      "args": ["-y", "@temporal-cortex/cortex-mcp"],
      "env": {
        "GOOGLE_CLIENT_ID": "your-client-id.apps.googleusercontent.com",
        "GOOGLE_CLIENT_SECRET": "your-client-secret"
      }
    }
  }
}
```

### Cursor

Add to Cursor's MCP settings (`~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "temporal-cortex": {
      "command": "npx",
      "args": ["-y", "@temporal-cortex/cortex-mcp"],
      "env": {
        "GOOGLE_CLIENT_ID": "your-client-id.apps.googleusercontent.com",
        "GOOGLE_CLIENT_SECRET": "your-client-secret"
      }
    }
  }
}
```

### Windsurf

Add to Windsurf's MCP config (`~/.codeium/windsurf/mcp_config.json`):

```json
{
  "mcpServers": {
    "temporal-cortex": {
      "command": "npx",
      "args": ["-y", "@temporal-cortex/cortex-mcp"],
      "env": {
        "GOOGLE_CLIENT_ID": "your-client-id.apps.googleusercontent.com",
        "GOOGLE_CLIENT_SECRET": "your-client-secret"
      }
    }
  }
}
```

> **Need Google OAuth credentials?** See [docs/google-cloud-setup.md](docs/google-cloud-setup.md) for a step-by-step guide.

## First-Time Setup

On first run, the server needs Google Calendar access. You have two options:

**Option A** — Run auth before connecting:
```bash
npx @temporal-cortex/cortex-mcp auth
```
This opens your browser for Google OAuth consent. After authorizing, credentials are saved to `~/.config/temporal-cortex/credentials.json` and reused automatically.

**Option B** — The server prompts automatically when an MCP client connects and no credentials are found.

After authentication, verify it works by asking your AI assistant: *"What meetings do I have today?"*

## Available Tools

| Tool | Description | What Makes It Different |
|------|-------------|------------------------|
| `list_events` | List calendar events in a time range | Output in TOON format (~40% fewer tokens) or JSON |
| `find_free_slots` | Find available time slots in a calendar | Computes actual gaps between events, not just event data |
| `book_slot` | Book a calendar slot safely | Lock → verify → write with Two-Phase Commit. No double-bookings. |
| `expand_rrule` | Expand a recurrence rule into concrete instances | Deterministic: handles DST, BYSETPOS, leap years, EXDATE |
| `check_availability` | Check if a specific time slot is available | Verifies against both events and active booking locks |
| `get_availability` | Unified availability across multiple calendars | Merges busy/free data with privacy controls (opaque/full) |

See [docs/tools.md](docs/tools.md) for full input/output schemas and usage examples.

## The RRULE Challenge

Most AI models and calendar tools silently fail on recurrence rule edge cases. Run the challenge to see the difference:

```bash
npx @temporal-cortex/cortex-mcp rrule-challenge
```

### 5 cases where LLMs consistently fail

**1. "Third Tuesday of every month" across DST (March 2026, America/New_York)**

The third Tuesday is March 17. Spring-forward on March 8 shifts UTC offsets from -05:00 to -04:00. LLMs often produce the wrong UTC time or skip the month entirely.

**2. "Last Friday of every month" (BYSETPOS=-1)**

`RRULE:FREQ=MONTHLY;BYDAY=FR;BYSETPOS=-1` — LLMs frequently return the first Friday instead of the last, or fail to handle months with 4 vs 5 Fridays.

**3. "Every weekday except holidays" (EXDATE with timezone)**

`EXDATE` values with explicit timezone offsets require exact matching against generated instances. LLMs often ignore EXDATE entirely or apply it to the wrong date.

**4. "Biweekly on Monday, Wednesday, Friday" (INTERVAL=2 + BYDAY)**

`RRULE:FREQ=WEEKLY;INTERVAL=2;BYDAY=MO,WE,FR` — The `INTERVAL=2` applies to weeks, not individual days. LLMs frequently generate every-week occurrences instead of every-other-week.

**5. "February 29 yearly" (leap year recurrence)**

`RRULE:FREQ=YEARLY;BYMONTH=2;BYMONTHDAY=29` — Should only produce instances in leap years (2028, 2032...). LLMs often generate Feb 28 or Mar 1 in non-leap years.

Truth Engine handles all of these deterministically using the [RFC 5545](https://www.rfc-editor.org/rfc/rfc5545) specification. No inference, no hallucination.

## How It Works

The MCP server is a single Rust binary distributed via npm. It runs locally on your machine and communicates with MCP clients over stdio (standard input/output).

- **Truth Engine** handles all date/time computation: RRULE expansion, availability merging, conflict detection. Deterministic, not inference-based.
- **TOON** (Token-Oriented Object Notation) compresses calendar data for LLM consumption — fewer tokens, same information.
- **Two-Phase Commit** ensures booking safety: acquire lock, verify the slot is free, write the event, release lock. If any step fails, everything rolls back.

### Lite Mode vs Full Mode

Mode is auto-detected — there is no configuration flag.

- **Lite Mode** (default): No infrastructure required. Uses in-memory locking and local file credential storage. Designed for individual developers with a single Google Calendar account.
- **Full Mode** (activated when `REDIS_URLS` is set): Uses Redis-based distributed locking (Redlock) for multi-process safety. Designed for production deployments with multiple concurrent agents.

## Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GOOGLE_CLIENT_ID` | Yes* | — | Google OAuth Client ID from [Cloud Console](https://console.cloud.google.com/apis/credentials) |
| `GOOGLE_CLIENT_SECRET` | Yes* | — | Google OAuth Client Secret |
| `GOOGLE_OAUTH_CREDENTIALS` | No | — | Path to Google OAuth JSON credentials file (alternative to `CLIENT_ID` + `CLIENT_SECRET`) |
| `REDIS_URLS` | No | — | Comma-separated Redis URLs. When set, activates Full Mode with distributed locking. |
| `TENANT_ID` | No | auto-generated | UUID for tenant isolation |
| `LOCK_TTL_SECS` | No | `30` | Lock time-to-live in seconds |
| `OAUTH_REDIRECT_PORT` | No | `8085` | Port for the local OAuth callback server |

\* Either `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET`, or `GOOGLE_OAUTH_CREDENTIALS` must be set.

See [docs/google-cloud-setup.md](docs/google-cloud-setup.md) for a complete setup guide.

## Going to Production?

The MCP server in Lite Mode handles single-provider computation with conflict prevention for individual use. For teams and production deployments:

- **Multi-provider unification** — Google + Outlook + CalDAV (iCloud, Fastmail) in one query
- **Distributed Two-Phase Commit** — Redis Redlock quorum for multi-process, multi-host safety
- **Enterprise OAuth management** — HashiCorp Vault integration, per-tenant credential isolation
- **Usage metering** — Per-operation billing with daily aggregation

| Operation | Price |
|-----------|-------|
| Calendar read | $0.001 |
| Availability check | $0.002 |
| Booking (with 2PC safety) | $0.01 |
| Connected account | $0.50/mo |
| **Free tier** | **100 bookings/mo + 5 accounts** |

No credit card required.

**[Request Platform Early Access](https://tally.so/r/aQ66W2)**

## Comparison with Alternatives

| Feature | temporal-cortex-mcp | google-calendar-mcp (nspady) | calendar-mcp (rauf543) |
|---------|:-------------------:|:----------------------------:|:----------------------:|
| Double-booking prevention (2PC) | Yes | No | No |
| Deterministic RRULE expansion | Yes | No | Partial |
| Multi-calendar availability merge | Yes | No | No |
| Prompt injection firewall | Yes | No | No |
| TOON token compression | Yes | No | No |
| Multi-provider (Google + Outlook) | Roadmap | No | No |
| Price | Free (Lite) | Free | Free |

## Built on Temporal Cortex Core

The computation layer is open source:

- **[temporal-cortex-core](https://github.com/billylui/temporal-cortex-core)** — Truth Engine (RRULE expansion, availability merging) + TOON (token compression)
- Available on [crates.io](https://crates.io/crates/truth-engine), [npm](https://www.npmjs.com/package/@temporal-cortex/truth-engine), and [PyPI](https://pypi.org/project/temporal-cortex-toon/)
- 446+ Rust tests, 42 JS tests, 30 Python tests, ~9,000 property-based tests

## Contributing

Bug reports and feature requests are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
