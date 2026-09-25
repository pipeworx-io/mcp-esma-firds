# @pipeworx/esma-firds

ESMA's EU-wide financial instrument reference register (FIRDS) and its MiFID II investment-firm authorisation register (the Union Register) — resolve an ISIN to its issuer LEI, CFI code and every EU trading venue it lists on; search instruments by issuer; and check whether a firm is authorised to provide investment services in the EU. Keyless, live per-query against ESMA's own public Solr endpoint.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `firds_lookup(isin)` — resolve one ISIN to full name, short name, CFI code, issuer LEI, notional currency, bond fields (maturity, nominal value, fixed rate) or derivative fields (underlying ISIN, expiry, option type) when applicable, and every EU trading venue (MIC) it trades on with each venue's status and first/termination trading date.
- `firds_search_by_issuer(issuer_lei?, issuer_name?, limit?)` — every instrument FIRDS holds for one issuer, by LEI (exact) or name (all words must appear, order-independent).
- `esma_check_firm_authorisation(name?, lei?, limit?)` — look up a firm in ESMA's Union Register of investment firms: authorisation status, competent authority, home member state, entity type.

## Auth

Keyless.

## Data sources

- `GET https://registers.esma.europa.eu/solr/esma_registers_firds/select` — the FIRDS Solr core: one document per (ISIN, trading venue) pair. Backs `firds_lookup` and `firds_search_by_issuer`.
- `GET https://registers.esma.europa.eu/solr/esma_registers_upreg/select` — the Union Register (MiFID II firm authorisation) Solr core, `entity_type:ae` documents. Backs `esma_check_firm_authorisation`.

Both cores are the same public, keyless Solr instance ESMA runs itself behind the FIRDS/register UI at registers.esma.europa.eu — confirmed live 2026-09-23. This pack calls it per request; it does not ingest ESMA's bulk FIRDS full/delta XML files (see Quirks).

## Quirks

- **Proxy, not mirror, on purpose.** ESMA also publishes FIRDS as downloadable full/delta XML files, and a naive build of this pack would ingest and index those locally. That would be redistribution and would need a reuse grant under Bruce's fleet #1389 ruling. The Solr endpoint above serves the identical reference data per query, keyless, so this pack calls it live instead — no bulk file was ever downloaded or stored.
- **One ISIN is many rows.** FIRDS files a separate record per trading venue, so a single well-listed share (e.g. SAP SE, `DE0007164600`) returns ~90 rows. `firds_lookup` always adds `AND latest_received_flag:1` to the query to get the *current* state per venue rather than FIRDS' full change-log history — omitting that flag silently mixes stale/superseded rows in with current ones. It then picks the record where `mic == rca_mic` (the venue ESMA itself treats as the reference market) for the descriptive fields (name, CFI code, currency, LEI), falling back to the first non-terminated (`status` not `TERM`/`CANC`) row, and folds every row into the `venues` list.
- **Name fields are un-tokenized strings, not full-text.** `gnr_full_name` (FIRDS) and `ae_entityName` (Union Register) each hold the whole name as one string value — a wildcard only matches from the start of that value, so `Deutsche* AND Bank*` matches nothing (no value starts with two different prefixes). `firds_search_by_issuer`'s `issuer_name` and `esma_check_firm_authorisation`'s `name` instead split the input on whitespace and AND together a `*word*` substring clause per word, which correctly requires every word to appear somewhere in the value regardless of order. Matching is case-insensitive (verified: `sap*` and `SAP*` return the same set).
- **`mrkt_trdng_trmination_date`** — FIRDS' own field name is missing the "e" (not a typo introduced here); kept as-is since it's the literal Solr field name.
- The Union Register core name is `esma_registers_upreg` (Union Register), not an obvious "mifid" or "entities" name — found by brute-forcing likely core names against `/solr/<core>/select`, since ESMA documents no core-name list. A sibling core, `esma_registers_fitrs_equities` (MiFIR transparency/liquidity data — ADT, LIS thresholds), exists on the same host and is a plausible future addition but is out of scope here.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "esma-firds": {
      "url": "https://gateway.pipeworx.io/esma-firds/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/esma-firds/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/firds_lookup \
  -H 'Content-Type: application/json' \
  -d '{"isin":"DE0007164600"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/firds_lookup`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "esma-firds": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-esma-firds"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-esma-firds
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Esma Firds data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
