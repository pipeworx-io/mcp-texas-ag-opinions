# @pipeworx/texas-ag-opinions

Texas Attorney General opinions since 1939: look up a specific opinion by
number (full text, requestor, cited statutes), check whether it has a
recorded overruled/modified/affirmed/withdrawn status, and search that status
index by statute citation.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1764+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `tx_ag_opinion_lookup(opinion_number)` — the AG, date, requestor, RQ-
  request number, topic categories, one-sentence summary, full opinion text,
  and statute citations found in it (best-effort regex over the opinion PDF).
  Also includes the opinion's own subsequent-treatment status if it has one.
- `tx_ag_opinion_status(opinion_number)` — whether a specific opinion has a
  recorded overruled/modified/affirmed/superseded/withdrawn entry in the AG's
  own courtesy index. `not_listed` does NOT mean the opinion is still good
  law — the AG's own page says the list may be incomplete.
- `tx_ag_statute_search(citation)` — search the ~381 opinions with a recorded
  subsequent-treatment entry for one mentioning a given statute citation or
  keyword. PARTIAL CAPABILITY — see "Data sources" below for exactly what
  this does and does not cover.

## Auth

Keyless.

## Data sources

- <https://www.texasattorneygeneral.gov/opinions> — landing page listing all
  16 Attorneys General since 1939 with their exact opinion-number range (e.g.
  "GA-0001 - GA-1096"). Scraped once (2026-10-07) into `TX_AG_RANGES` in
  `src/index.ts` — this is how `tx_ag_opinion_lookup` and `tx_ag_opinion_status`
  turn a bare number into the right AG slug without guessing.
- <https://www.texasattorneygeneral.gov/opinions/{ag-slug}/{number}> — one
  opinion's own page (e.g. `/opinions/ken-paxton/kp-0537`): AG name, topic
  categories, one-sentence summary, and a link to the opinion PDF. **Fetched
  live**, never baked.
- <https://www.texasattorneygeneral.gov/sites/default/files/opinion-files/opinion/{year}/{number}.pdf>
  — the opinion's full text. Verified text-based (not scanned): `extractPdfText`
  (shared/src/pdf-text.ts) parses it cleanly with zero warnings. Requestor
  name/title/address, the RQ- request number, the one-line question, and cited
  statutes are all regex-extracted from this text — best-effort, not a real
  citation parser (a multi-section cite like "§§ 51.031, .034" only captures
  the first section; non-"TEX. ... CODE §" citation styles, e.g. to federal
  law or by "Art." instead of "§", are not captured).
- <https://www.texasattorneygeneral.gov/opinions/opinions-overruled-modified-affirmed-withdrawn>
  — ONE page listing every opinion (381, captured 2026-10-07) the AG's Opinion
  Committee has flagged as later overruled, modified, affirmed, superseded, or
  withdrawn, with the citing statute or case. Baked into
  `src/tx-ag-status-data.ts` (~38KB — comfortably under the few-hundred-KB
  threshold that would require KV baking, so this stays a plain module-scope
  import, unlike the state-code packs with much larger tables). This index is
  what backs both `tx_ag_opinion_status` and `tx_ag_statute_search`.

### What this pack cannot do, and why

**`/opinions/{number}` WITHOUT the AG slug is a trap.** It returns HTTP 200
for ANY number, real or fabricated — verified live: a real opinion (kp-0537)
and a made-up one (kp-9999) come back byte-for-byte the same size, because
the slug-less path is a generic "ajax-individual-opinion" form stub that says
"Invalid Attorney General name in URL", not the opinion. The correctly
AG-qualified path 404s properly on a bad number. This pack always resolves
the AG slug from `TX_AG_RANGES` first and never fetches the slug-less path.

**Full-text statute-citation search across the whole archive (~19,000
opinions back to 1939) is NOT available.** The site's own opinion search box
is a Cludo widget (`api-us1.cludo.com`, customerId 10000129, engineId
10001310) that renders results entirely client-side after a signed API call —
confirmed live: every call shape tried (GET, POST, with/without Origin and
Referer headers) returns `401 {"Message":"Authorization has been denied for
this request."}`. `tx_ag_statute_search` instead searches only the baked
overruled/modified/affirmed/withdrawn index (381 opinions) for a citation or
keyword appearing in the text describing what superseded each one — useful
for "has this been overruled by a later statute," not for "which opinions
discuss this statute" in general. Every response from that tool states this
limitation in a `coverage_note` field.

**Cloudflare JS challenge under load.** The site is Cloudflare-fronted and
answered a "Just a moment..." challenge page (not the real content) to one of
this pack's own research requests after ~15 rapid sequential fetches. Both
live-fetching tools detect this (`isCloudflareChallenge`) and throw a loud
`upstream_down:` error naming it, rather than silently parsing the challenge
page as if it were real content.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "texas-ag-opinions": {
      "url": "https://gateway.pipeworx.io/texas-ag-opinions/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/texas-ag-opinions/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1764+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/tx_ag_opinion_lookup \
  -H 'Content-Type: application/json' \
  -d '{"opinion_number":"KP-0537"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/tx_ag_opinion_lookup`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "texas-ag-opinions": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-texas-ag-opinions"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-texas-ag-opinions
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Texas Ag Opinions data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
