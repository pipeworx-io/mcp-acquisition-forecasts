# acquisition-forecasts

US federal agency **procurement forecasts** — what an agency has published its
intention to buy, months to years before any solicitation exists. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

This is the forward-looking surface. `usaspending` is money already obligated;
`govcon-intel` scans SAM for solicitations already open. A forecast is the only
one that answers *"should I be staffing for this"* rather than *"did I miss it"*.

## Tools

- `forecast_search(keyword?, agency?, sub_agency?, naics?, set_aside?, fiscal_year?, min_value?, limit?)`
- `forecast_detail(forecast_id, agency?)` — full narrative, PoP, contacts
- `forecast_agencies()` — what is wired, and the verdict for everything that isn't

## Coverage is one agency, and that is the finding

Agencies are required to publish forecasts and acquisition.gov indexes 22 of
them. I probed **every one** on 2026-08-29. Exactly **one** publishes machine-
readable data:

| Verdict | Count | Agencies |
|---|---|---|
| **JSON API** | **1** | **DHS** (apfs-cloud.dhs.gov, 847 rows) |
| HTML, no data file | 15 | Agriculture, Education, Energy, Justice, Labor, State, Interior, Treasury, Transportation, VA, GSA, NASA, NSF, SSA, DHS's own landing page |
| PDF only | 4 | **DoD**, HUD, NRC, OPM |
| Blocks automated clients (403) | 2 | USAID, Commerce |
| Unreachable (timeout) | 1 | HHS |

PDF forecasts are **deliberately not OCR-ed**. A parsed number that is wrong is
worse than no number, and these are dollar values people plan against.

Because of that, `forecast_agencies()` ships as a first-class tool and
`agencies_covered` + `coverage_note` ride on every search response. A pack named
for federal forecasts that quietly covered one agency would answer *"nothing
planned"* for 21 agencies it never checked — the absence-of-data-vs-absence-of-
the-thing failure, which is the one this catalog keeps re-learning.

Adding an agency is a config entry in `AGENCIES[]` plus a field map.

## Traps

**The APFS API ignores every query parameter.** `?naics=5415`,
`?organization=CBP` and `?zzznonsense=x` all return the identical 847 rows with
the identical first id. There is no server-side filtering, so the pack fetches
the whole file (~2 MB, cached 15 min) and filters in memory. Anyone
"optimising" this by pushing filters upstream gets a result that looks filtered
and is not.

**`small_business_set_aside` and `small_business_program` are different fields.**
The first is a **boolean** (true/false/null), the second is the program **name**
(`SB`, `8(a)`, `HUBZone`, `WOSB`, `SDVOSB`, `AbilityOne`, `None`, `TBD`).
Coalescing the boolean first turns 350 of 847 rows into the literal string
`"true"` and costs them their program name — and makes `set_aside="8(a)"` return
**27 rows instead of 71**, because only rows whose boolean happened to be null
could still match. They are returned as `set_aside` (the name) and
`set_aside_applies` (the boolean, `null` meaning *the row does not say*).

**NAICS arrives as code *and* label in one string** — `"236220 - Commercial and
Institutional Building Construction"`. Matching a caller's `5415` against the raw
field would also hit any description containing those digits. The code is split
out and matched as a **prefix**, which is how NAICS hierarchy works: `5415`
covers `541511`, `541512`, … (175 rows vs 19 for the full code).

**Values are bands, not numbers** — `"$50M to $100M"`, `"Over $100M"`. The band
text is preserved verbatim *and* parsed to `low`/`high`. For an open-ended top
band `high` is **null**, not a large number, and `min_value` treats it as
unbounded — a fabricated ceiling would be sortable and wrong.

**`organization` is a sub-agency path**, not an agency: `CBP/OIT/LSS/ITB`. All
847 rows are DHS. `agency` and `sub_agency` are reported separately so
`agency="DHS"` means what a caller expects.

**Forecasts are revised.** `published_date`, `previously_published` and
`last_updated` say which version you are looking at; a withdrawn or awarded
requirement can drop out of the file entirely, which is why `forecast_detail`
says so rather than just reporting "not found".

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "acquisition-forecasts": {
      "url": "https://gateway.pipeworx.io/acquisition-forecasts/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/acquisition-forecasts/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/forecast_search \
  -H 'Content-Type: application/json' \
  -d '{"naics":"5415","agency":"DHS"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/forecast_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-acquisition-forecasts` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/acquisition-forecasts/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "acquisition-forecasts": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-acquisition-forecasts"]
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
ask_pipeworx({ question: "your question about Acquisition Forecasts data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
