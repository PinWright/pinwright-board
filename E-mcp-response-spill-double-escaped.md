---
id: E-mcp-response-spill-double-escaped
title: "When an oversized reader spills, the written file is a double-escaped MCP text envelope, not the clean payload — naive grep/jq/Read fail and cost extra discovery calls"
status: OPEN
severity: Low
category: ergonomic
tags: [spill-file-double-escaped, response-size, oversized-readback, mcp-envelope, docs]
encounters: 1
lastSeen: 2026-07-01T22:37:09.9601693+03:00
---

# Oversized-response spill file is a double-escaped MCP envelope, not the payload JSON

When any large reader overflows the display limit, the payload lands in a file the
caller then Reads. But that file is **not** the clean JSON payload — it is the MCP
transport envelope with the payload JSON-**escaped inside a text field**:

```json
{"content":[{"type":"text","text":"{\r\n\t\"objects\": [ ... \"class\": \"WorldSettings\" ... }"}]}
```

The real content is `\"class\": \"...\"` (escaped quotes + spaces + `\r\n\t`
whitespace escaping), so the caller's natural tooling breaks:

- `Grep '"class":"[^"]*"'` → **No matches found** (the file has `\"class\": \"…\"`,
  not `"class":"…"`), even though `Grep 'class'` on the same file returns 228 hits.
- A plain `Read` of the file failed — *"39593 tokens exceeds maximum allowed
  25000"* — because the escaped `\r\n\t` bloats the byte count.

So recovering the payload cost **two extra probing calls** — a count-only Grep, then
a PowerShell `.Substring(0,600)` — just to reverse-engineer the envelope shape
before the agent could parse it.

## Sanity-check on the suspect method

The CallAnalyzer attributed this to `system.inspect.list_objects`, but that method
is only the *trigger* — the friction is in the **spill-file format**, which is
identical for every reader that overflows (`inspect_object`, `list_objects`,
`asset.search`, the whole `*-no-limit-spills` family). This is a cross-cutting
transport-envelope ergonomic, not a `list_objects` defect, and it is **not** the
same as the per-method narrowing tickets (which reduce how *often* a reader spills;
this is about the spilled file being unparseable *when* it does).

## What it should do

Make the spilled/oversized payload machine-parseable without reverse-engineering:

- Prefer emitting the payload as MCP **`structuredContent`** (the actual JSON
  object) rather than a JSON string stuffed into a `text` block, so a spilled tool
  result exposes the payload directly instead of double-escaped inside `text`.
- Where a text envelope is unavoidable, keep the payload **compact** (no `\r\n\t`
  pretty-print inside the escaped string) so the file stays under Read limits and
  grep patterns match on `"class":`.
- **Docs (`docs/wiki-src/system.inspect.md` overlay + the wiki spill note):**
  document that a spilled response file is the MCP `{content:[{type:text,text:…}]}`
  envelope with the payload **JSON-escaped inside `text`**, and give the working
  recovery recipe (parse `.content[0].text` then JSON-decode; grep `\"class\":`
  not `"class":`), so callers don't burn discovery calls learning the shape.

Caveat (for the downstream dev): on the MCP path pinwright deliberately bypasses
its own server-side spill (`E-http-response-spill`: "MCP clients own large-output
behavior"), so the *file itself* is written by the MCP client. The pinwright-side
lever is therefore the response **shape** (structuredContent / compact) that
determines how parseable the client's spill is — not the file-writing code.

## Evidence

From a realism-mode orientation task (focus `null`, namespace `system`, outcome
**done** — every call first-try). CallAnalyzer call-trace finding, verbatim:

> "the file it wrote is the MCP TRANSPORT ENVELOPE with the payload JSON-escaped
> inside a text field ({"content":[{"type":"text","text":"{\r\n\t\"objects\":
> [...\"class\": \"WorldSettings\"...}"}]}), not the clean JSON payload. The
> agent's natural Grep pattern '"class":"[^"]*"' returned 'No matches found' …
> and a plain Read of the file failed ('39593 tokens exceeds maximum allowed
> 25000'). Agent needed 2 extra probing calls (a count Grep, then a PowerShell
> .Substring(0,600)) before it could parse."

Fully recoverable (the data is there, just awkward to extract), hence Low.

## Distinct from

- `E-http-response-spill` (DONE) — that ticket built the *server-side HTTP* spill
  mechanism and writes a **clean** raw JSON-RPC result to disk for direct `/rpc`
  callers; this ticket is that the **MCP-path** spill (client-written) is the
  double-escaped text envelope, which the HTTP mechanism explicitly bypasses.
- `E-inspect-list-objects-no-limit-spills` / `E-inspect-object-no-projection-spills`
  / the `*-no-limit-spills` family — those reduce how *often* a specific reader
  overflows (add filter/limit/projection); this is orthogonal: the format of the
  file produced *when* an overflow spills. Even a fully-narrowed reader, asked for
  the full set, spills the same double-escaped envelope.

severity rationale: impact=pure-friction (payload recoverable via a Read + shape
discovery, no data lost) × reach=fires on any oversized reader across many
sessions (bump-eligible, but fully worked around) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a realism-mode orientation task (focus `null`, namespace `system`, outcome **done**; every call first-try). PROCESS friction: when `system.inspect.list_objects {}` overflowed (108306 chars), the spill file was the MCP transport envelope `{content:[{type:text,text:"<json-escaped payload>"}]}` — not the clean payload — so the agent's `Grep '"class":"[^"]*"'` returned "No matches found" (real content is `\"class\": \"…\"`) while `Grep 'class'` hit 228 times, and a plain `Read` failed ("39593 tokens exceeds maximum allowed 25000"), costing 2 extra probing calls (count Grep + PowerShell `.Substring(0,600)`) to reverse-engineer the shape. Sanity-checked the CallAnalyzer's `list_objects` suspect: the real subject is the cross-cutting spill-envelope format (identical for `inspect_object`/`asset.search`/every `*-no-limit-spills` reader), so filed against the response shape, not `list_objects`. Proposes emitting `structuredContent` (raw JSON object) and/or compact (non-pretty) payload so a spilled MCP result is machine-parseable, plus a wiki note documenting the `.content[0].text` recovery recipe. Caveat noted: on the MCP path pinwright bypasses its own server-side spill (`E-http-response-spill`), so the file is client-written and the pinwright lever is the response shape. Dedup: ripgrep across OPEN/DONE/WONTFIX — no ticket owns the spilled-file *format* (`E-http-response-spill` writes a clean file on the HTTP path it does not bypass; the `*-no-limit-spills` family reduces overflow frequency, not the spill format). Distinct family tag `spill-file-double-escaped`.
