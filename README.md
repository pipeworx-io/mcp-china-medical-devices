# @pipeworx/china-medical-devices

China's medical-device Unique Device Identification (UDI) register — search by product/trade
name or model, resolve one device identifier, look up a manufacturer, and list what changed
since a date, refreshed from the publisher's release files.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1861+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `udi_search(query, manufacturer?, limit?)` — free-text search over product name, trade name,
  and model (all terms must match), optionally filtered by registrant/manufacturer name.
- `udi_resolve(udi_di)` — exact lookup of one product-level UDI-DI.
- `udi_manufacturer(name?, registrant_id?, limit?)` — registrants matching a name or unified
  social credit code, with how many device identifiers and registration numbers each holds.
- `udi_recent_changes(since, limit?)` — devices whose latest published version is on or after a
  date, newest first, drawn from the publisher's delta release files.
- `udi_coverage()` — device count, newest/oldest dates, and whether the search indexes are
  built yet. Check this before trusting `udi_search` latency on a very broad term.

## Auth

Keyless. No vendor key is involved — the data lives in Pipeworx's own store, loaded from the
publisher's own release files.

## Data sources

- NMPA (National Medical Products Administration), Unique Device Identification database —
  the official register of medical-device Unique Device Identifiers for devices sold in China.
  The publisher distributes this as periodic bulk release files (a full release plus delta
  files), not a public query API, so this pack is refreshed on that cadence
  rather than calling the publisher per request. The exact release/delta routes live only in
  `src/index.ts` and `scripts/nmpa-udi-load.py`.

**Licence note:** the publisher's download page carries only a copyright line
(版权所有：国家药品监督管理局); no explicit reuse terms were found on it or in the release files
themselves. Under the standing rule for a source that only publishes bulk files and is silent
on reuse (hold only when terms *forbid* redistribution; build and flag when they are silent),
this pack ships with that flag rather than a hold. If explicit terms forbidding redistribution
turn up later, this needs to be revisited.

**What is deliberately not stored:** registrants' contact details (e-mail, phone) the release
carries — personal data this pack has no reuse ruling to serve — and packaging/carton-level
identifiers, a different register at a different grain nobody has asked for. A lookup for one
of those resolves `found: false` with a note, not an error.

**Demand status:** this pack was built from a standing Bruce ask to assess China-source
coverage (2026-10-21), not from measured demand — the 30-day miss log showed zero misses for
this specific register at the time it was proposed. Build-to-see-what-it-does, not a backlog
item that real traffic was waiting on.

**Cross-link:** `@pipeworx/china-pharma` covers NMPA *drug* approvals and policy announcements
(via HKEX disclosure filings and the NMPA English-language announcements) — a different register, drugs rather
than devices, at a different grain. Use that pack for drug approvals; use this one for device
identifiers.

## Data and refresh

The register is loaded from the publisher's full release and
kept current by applying its delta files (daily, weekly, monthly) in date order. The loader is
resumable and idempotent: a stored device is only replaced by a version that is not older.
Every response carries `data_as_of` (the release date of the latest file applied) and, for
`udi_resolve` / `udi_coverage`, `ingested_at`.

**Individuals are not served.** Some registrants are sole proprietors filing in their own name
(the published registrant name is just a personal name). Their registrant name, English name
and credit code are withheld at load and never stored; the device record itself (name, model,
registration number) is served. `udi_manufacturer` therefore lists companies only.

**Search speed.** `udi_search` and `udi_manufacturer` scan the table until the search
indexes are built (a separate database step run after the bulk load); until then expect
roughly 10-15 seconds on a broad term.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "china-medical-devices": {
      "url": "https://gateway.pipeworx.io/china-medical-devices/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/china-medical-devices/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1861+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/udi_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"起搏器","manufacturer":"美敦力","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/udi_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-china-medical-devices` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/china-medical-devices/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "china-medical-devices": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-china-medical-devices"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about China Medical Devices data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
