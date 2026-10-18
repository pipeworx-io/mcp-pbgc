# PBGC — pension plan terminations and premium filings

Whether a named pension plan was terminated and taken over by the federal
pension insurer (trusteed), or is still an active plan currently paying its
premiums — single-employer and multiemployer, from the regulator's own
registers.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1837+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Why this exists alongside `form5500`

`form5500` shows what a plan FILED with the Department of Labor each year —
sponsor, participants, assets. It does not say what happened to the plan
afterward. This pack answers that: did PBGC take it over because it
terminated underfunded, or is it still active and paying premiums. The two
registers (trusteed vs. premium-filer) are **disjoint by construction** — a
plan is in exactly one of PBGC's own rosters, never both at once.

## Tools

| Tool | Answers |
|---|---|
| `pbgc_plan_status` | *Did [sponsor]'s pension plan get taken over by PBGC?* — searches both registers and reports whichever matched |
| `pbgc_search_trusteed_plans` | *Pension plans PBGC has taken over in Illinois* |
| `pbgc_search_premium_filers` | *Pension plans sponsored by Delta Air Lines* — still active |
| `pbgc_industry_totals` | *How many pension plans has PBGC taken over?* — national or per state |

## Auth

Keyless.

## Data sources

- Pension Benefit Guaranty Corporation (PBGC), trusteed-plan register — single-employer
  pension plans PBGC has taken over as trustee because the plan terminated underfunded.
- Pension Benefit Guaranty Corporation (PBGC), premium-filer rosters — single-employer and
  multiemployer pension plans currently paying PBGC premiums (not terminated).

PBGC publishes these as periodically-updated open-government downloads rather
than through a query API — its own plan-search page is server-rendered HTML
with no JSON backend behind it. Routes live only in `scripts/ingest-pbgc.mjs`.

## Why this is a mirror, and the replace semantics that matter

PBGC distributes each register as a **complete current snapshot**, not a
delta — there is no "what changed since" feed. `scripts/ingest-pbgc.mjs`
(scheduled weekly by `.github/workflows/pbgc-refresh.yml`) stamps every row it
writes with that run's `data_as_of` and then deletes any row in the same
table left over from a previous run. That matters specifically for premium
filers: a plan that stops paying premiums (because it terminated, merged, or
PBGC took it over) has to actually disappear from `pbgc_search_premium_filers`
and `pbgc_plan_status`, not linger reading as still-active forever.

Every tool response carries `data_as_of` for exactly this reason — the two
registers refresh on different clocks (the trusteed register roughly
quarterly, the premium-filer rosters roughly monthly), so a caller comparing
a plan's status across both should know which snapshot each half came from.

## Schema

`supabase/migrations/262_pbgc.sql` — `pbgc_trusteed_plans` (one row per
trusteed case; this table only grows, since PBGC's own register does) and
`pbgc_premium_filers` (one row per currently-filing plan, `plan_type` +
`plan_id` as the key, replaced wholesale on each refresh).

## Scale (verified against the live source, 2026-10-17)

| | |
|---|---|
| Trusteed plans | ~5,180 single-employer cases since PBGC's creation in 1974 |
| Premium filers — single-employer | ~21,400 active plans |
| Premium filers — multiemployer | ~1,290 active plans |

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "pbgc": {
      "url": "https://gateway.pipeworx.io/pbgc/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/pbgc/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1837+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/pbgc_plan_status \
  -H 'Content-Type: application/json' \
  -d '{"sponsor":"US Airways"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/pbgc_plan_status`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "pbgc": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-pbgc"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-pbgc
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Pbgc data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
