---
id: E-mcp-transport-page-oversized
title: "wiki-src/mcp-transport.md is ~42 KB, twice the ~20,000-character page guideline, after today's proxy-tool sections; extract them to topic pages"
status: OPEN
severity: Low
category: ergonomic
tags: [wiki, docs, mcp-transport, page-size, proxy, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
---

# mcp-transport.md is over the page budget

Plugin `CLAUDE.md:320` sets a soft guideline of ~20,000 characters per wiki page and rule 4 (`:342`)
says to extract self-contained editorial blocks over 2 KB into `docs/wiki-src/<ns>.<topic-slug>.md`
topic pages. `docs/wiki-src/mcp-transport.md` is 41,931 characters in the working tree (31,795 at plugin
HEAD `71c91649`), so today's changes added ~10 KB. The largest sections are the stdio-proxy material:
`## Stdio proxy lifecycle tools` (~6.3 KB), `## Streaming responses (SSE)` (~4.2 KB),
`## Test runs: editor_run_tests, then editor_test_status` (~3.8 KB), `## Endpoint and port resolution`
(~3.4 KB) and `## Building: editor_build, then editor_build_status` (~2.7 KB).

The guideline calls an over-budget page a quality smell, not an error, and 40 `wiki-src` pages exceed
it (`render.md` 118 KB is the largest); this ticket covers only `mcp-transport.md` because it is the
entry page for the proxy tools agents use every session and it doubled today.

**Impact:** readability and grep friction on a frequently read page. No behaviour change.
**Fix:** move the three proxy-tool sections (lifecycle tools, test runs, building) into one topic page,
e.g. `docs/wiki-src/mcp-transport.proxy-tools.md`, and leave a two-line pointer plus a `## See also`
entry in `mcp-transport.md`. Confirm the page renders under ~20,000 characters.

## History
- `#1-page-over-budget` `OPEN` reporter — Measured today: 41,931 characters (`wc -m`), section sizes as above; 31,795 at HEAD.
