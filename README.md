# @pipeworx/morocco-hcp-statistics

Morocco's national economic and social indicators from the Haut-Commissariat au Plan (HCP), the national statistics authority: monthly and quarterly series for consumer prices (national and by city), industrial producer prices, industrial, mining, energy and fishing output, tourism nights, rail traffic, external trade, exchange rates, public finances, the Casablanca stock exchange, labour-force indicators, land registrations and national accounts. Labels are in the publisher's French wording.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1861+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ma_bulletins()` — the quarterly bulletins covered (title, quarter, publication date, series parsed from each, sheets not parsed), the topic areas, and `data_as_of`.
- `ma_indicators(search?, topic?, limit?)` — find indicator series by name. Accepts French terms ("indice general", "tourisme", "chomage") or English ones ("inflation", "unemployment", "euro"). Returns an id, table, unit note, frequency and latest value per hit. No arguments lists the topics.
- `ma_indicator_series(id, from?, to?)` — the full time series for one id (YYYY-MM or YYYY-Qn periods), the latest value, and the change versus the previous period and versus a year earlier (computed by Pipeworx from the published values and labelled as such). A miss returns `found:false` with `data_as_of` and nearby ids.

Every response carries `source`, `data_as_of` (the newest bulletin's publication date) and `reuse_terms`.

## Auth

Keyless.

## Data sources

- Haut-Commissariat au Plan (HCP, Morocco), Bulletin statistique — the quarterly statistical bulletin of national indicators.

The bulletin is published as a presentation-style workbook (bilingual French/Arabic, merged cells, several tables per sheet), not through a per-indicator interface. There is nothing to proxy, so this pack serves a parsed copy: `scripts/bake-index.mjs` reads the newest bulletins, finds each table's period header, turns every labelled numeric row into a series, merges the editions (newest wins per period) and writes `src/hcp-index-data.ts`. The gateway loads that table from KV on first use (`workers/gateway/src/pack-baked-indexes.json`); the pack makes no request to the publisher at call time.

### Reuse terms

The publisher's site terms (Conditions générales d'utilisation, Version 1.0) state:

> "En conformité avec les dispositions et normes internationales d'ouverture des données et notamment les dispositions de la licence CC BY 4.0, vous êtes autorisés à télécharger les indicateurs du site hcp.ma et bds.hcp.ma librement sauf spécification contraire et à les réutiliser, y compris à des fins commerciales. La réutilisation est toutefois conditionnée par le respect de l'intégrité de l'information et des données et la mention précise des sources."

Redistribution is therefore permitted with attribution (credit the Haut-Commissariat au Plan) and without altering the meaning of the data. Every response repeats this in `reuse_terms`.

### Refreshing

Run `node mcps/morocco-hcp-statistics/scripts/bake-index.mjs` when the publisher lists a new bulletin (about one per quarter; `--limit N` sets how many recent editions to merge, default 12). It needs the `unzip` CLI, makes one request per workbook, and refuses to write if the newest bulletin does not parse. Commit the regenerated `src/hcp-index-data.ts`; the deploy uploads it to KV.

### Known gaps

- **Ids are positional within a sheet.** When one label repeats in a sheet's table, later copies carry a `~N` suffix and are taken from the newest bulletin only, because their order cannot be trusted to line up across editions. A label spelled differently between editions (e.g. "1 EUR" and "1EUR") is held as two series.
- **Provisional and revised markers** the publisher puts in sheet headings (`*`, `R`) are stripped, not carried per value.
- **Units** come from the table heading (`unit_note`) and, for labour-force rows, from the row label ("en milliers"); there is no separate unit column in the source.
- **Not every sheet parses in every edition.** `ma_bulletins` lists the sheets that were not turned into series for each bulletin (for example the labour-force sheet in the Q1 2026 edition).
- **Older editions use older sheet layouts**; the merged history starts about three years back and is thinner for tables that were reorganised.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "morocco-hcp-statistics": {
      "url": "https://gateway.pipeworx.io/morocco-hcp-statistics/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/morocco-hcp-statistics/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/ma_bulletins \
  -H 'Content-Type: application/json' \
  -d '{}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ma_bulletins`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-morocco-hcp-statistics` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/morocco-hcp-statistics/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "morocco-hcp-statistics": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-morocco-hcp-statistics"]
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
ask_pipeworx({ question: "your question about Morocco Hcp Statistics data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
