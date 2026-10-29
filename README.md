# @pipeworx/hudoc

European Court of Human Rights (ECHR / Strasbourg) case law — 231,000+
judgments, decisions, advisory opinions, resolutions and communicated cases
across the 46 Council of Europe states, served live from the Court's own
HUDOC database.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `echr_search(query?, article?, respondent?, collection?, importance?, date_from?, date_to?, limit?)` — search case law by free text, Convention Article, respondent state and/or date. Returns case name, application number(s), Article(s), conclusion (violation/no violation) and the itemid to read the full text with.
- `echr_judgment(itemid, max_chars?, offset?)` — full text of one document by its HUDOC itemid, paged.

## Auth

Keyless. No registration, no API key.

## Data sources

- <https://hudoc.echr.coe.int/app/query/results> — the search API HUDOC's own web UI calls. Query syntax is a Lucene-ish field grammar (`contentsitename=ECHR AND (documentcollectionid2:JUDGMENTS) AND article=8 AND respondent=FRA AND kpdate>=2024-01-01 AND "surveillance"`). Free text is a bare double-quoted phrase with no field name — a field-qualified phrase like `conclusion="freedom of expression"` silently returns zero results.
- <https://hudoc.echr.coe.int/app/conversion/docx/html/body?library=ECHR&id=...> — the document body, converted from the Court's stored DOCX to HTML. This is NOT the `/eng?i=...` page URL: that URL is a static Angular SPA shell — same ~15.7KB byte-for-byte regardless of itemid, including nonexistent ones — and never contains the judgment text. The conversion endpoint is what the SPA itself calls client-side to render the document.
- Very recently published documents (seen: within ~2-4 days) can 204 on the conversion endpoint — the HTML conversion lags the search index. `echr_judgment` reports this explicitly (`found: false, reason: "not_yet_converted"`) rather than returning empty text silently; `echr_search`'s `conclusion` field is populated immediately regardless.
- French-language-only documents (`doctype` starting `HF`) frequently 204 on the same endpoint even when old; `echr_search` defaults to `languageisocode=ENG` and excludes `doctype=PR` (press-release stubs, which otherwise leak into every collection filter) to route around both traps.
- `respondent` takes an ISO3166-1 alpha-3 code (`FRA`, `GBR`, ...); the tool also accepts common country names via a small lookup table covering all 46 member states plus Russia (expelled 2022, but its pre-expulsion case law is still in HUDOC and still gets asked about).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "hudoc": {
      "url": "https://gateway.pipeworx.io/hudoc/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/hudoc/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/echr_search \
  -H 'Content-Type: application/json' \
  -d '{"article":"8","respondent":"GBR","date_from":"2024-01-01","date_to":"2026-12-31","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/echr_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-hudoc` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/hudoc/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "hudoc": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-hudoc"]
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
ask_pipeworx({ question: "your question about Hudoc data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
