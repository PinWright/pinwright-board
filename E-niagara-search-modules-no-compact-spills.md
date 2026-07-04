---
id: E-niagara-search-modules-no-compact-spills
title: "niagara.search_modules has no compact/path-only projection — a handful of matches carry full multi-line descriptions and empty inputs/outputs arrays, so discovery queries overflow the 10000-char inline limit and spill to file"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, search-modules, response-size, oversized-readback, compact, projection, node-discovery, docs]
encounters: 1
lastSeen: 2026-07-04T18:09:33.0624177+03:00
---

# `niagara.search_modules` results are too verbose to read one assetPath inline — the discovery search spills to disk

`niagara.search_modules` is the discovery RPC an author calls to find the module
asset path to hand to `niagara.add_module`. But its default projection returns,
for every matched module, the full multi-line (`\r\n`-laden) description **plus**
an empty `"inputs":[]` / `"outputs":[]` array per hit, so even a query that
matches only a handful of modules routinely crosses the **10000-char MCP inline
display threshold** and spills the full payload to a
`Saved/.../HttpResponses/<uuid>.json` file. The caller then has to do an extra
`Read`/`Grep` of the spill file just to extract the single `assetPath` (or
`name`) the search existed to produce.

In this task (build `/Game/FX/NS_CampfireEmbers` from scratch) nearly every
`search_modules` call spilled, each forcing a follow-up
`Grep '"(assetPath|name)":'` over the written file:

- `query="solve forces and velocity"` -> `outputTooLong`, **59759 chars**
- `query="color"` -> **22557 chars**
- `query="initialize"` `limit=15` -> **17875 chars**
- `query="spawn"` `limit=15` -> **16657 chars**

The byte dominator is per-hit: the long multi-line description string and the
always-present empty `inputs`/`outputs` arrays, so the overflow is driven by
result-set size x per-hit verbosity, not by anything the caller can pre-narrow
short of already knowing the exact leaf name (which defeats the point of a
search). Roughly 14 `search_modules` RPCs were made in this build, several of
which spilled and needed a spill-file Read — turning module lookup into a search
RPC + a disk round-trip apiece.

## What it should do (downstream fix, not mine)

Mirror the opt-in size knob the board already shipped/proposed for the sibling
verbose discovery searches — same family, per method:

- **`compact: true`** (default `false`, byte-identical for existing callers) —
  per result emit only `assetPath` + `name` + `usage` + `validStages`, dropping
  the fat multi-line `description` and the empty `inputs`/`outputs` arrays, so a
  discovery search that returns a handful of modules stays inline.
- Optionally a **`fields`/`namesOnly` allow-list projection** (name+path only)
  for the dominant "which module path do I add?" case.
- Optionally **lower the default `limit`** on the interactive/MCP path so a broad
  query doesn't return a full page of fully-expanded records by default; keep
  `totalMatches`/`truncated` so elision stays detectable.
- **Docs (`docs/wiki-src/niagara.search_modules.md`):** note that the default
  result carries full per-hit descriptions and can exceed the inline budget
  (spilling to a HttpResponses file), and that the compact/`fields` projection or
  a narrower `limit` keeps a discovery scan inline — so the overflow is a
  documented expectation, not a surprise on the tool's headline use.

## Distinct from

- `E-niagara-standard-stack-recipe-undocumented` (OPEN, judge-filed for THIS same
  task) — same method (`niagara.search_modules`) but the orthogonal **content /
  keyword-match** gap (multi-word phrases like `spawn rate`, `particle state`,
  `initialize particle` return 0/irrelevant results) plus the missing canonical
  stack recipe. That ticket is about *whether the query matches the right
  module*; this ticket is about *the matched result being too big to display
  inline*. Different root cause; the board files search content-miss and search
  response-size separately (cf. `E-metasound-shorthand-search-mismatch` vs
  `E-search-metasound-nodes-no-compact-mode`).
- `E-niagara-inspect-no-param-readback-projection` (IN-REVIEW) — same
  response-size family but a **different niagara method** (`niagara.inspect`, the
  whole-asset structural dump), not the module-registry discovery search.
- `E-search-metasound-nodes-no-compact-mode` (OPEN) — the exact same
  no-compact-mode/verbose-per-hit spill shape on the MetaSound node-discovery
  search; this is the Niagara-module-discovery member of that family, which the
  board tracks per method.
- `E-console-search-default-limit-spills` (IN-REVIEW) — same shape
  (fat per-row payload x default limit -> spill -> forced Read) on
  `system.console.search`; there the fat column is `help`, here it is the
  module `description` + empty `inputs`/`outputs`.
- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism;
  this ticket is a *specific verbose reader* overflowing by default, the same
  relationship the whole no-compact/no-limit family has to that mechanism.

## Evidence

PROCESS friction from a clean/done fuzz task (namespace `niagara`, built
`/Game/FX/NS_CampfireEmbers` end-to-end — system + emitter + sprite renderer +
EmitterState/SpawnRate/InitializeParticle/AddVelocity/ParticleState/Drag/
SolveForcesAndVelocity; compile + strict-validate + save all green; judge filed
nothing on the outcome). CallAnalyzer call-trace finding, verbatim:

> Nearly every search_modules result spilled past the 10000-char inline
> threshold and had to be recovered via an extra Read/Grep of the HttpResponses
> JSON just to read out one assetPath. Examples: query 'solve forces and
> velocity' -> outputTooLong 59759 chars; 'color' -> 22557; 'initialize'
> limit15 -> 17875; 'spawn' limit15 -> 16657 — each followed by a Grep for
> '(assetPath|name):' over the spill file. The result payload carries a long
> multi-line description plus empty inputs/outputs arrays per hit, which is what
> pushes even a handful of modules over threshold.

Agent friction note, verbatim:

> "most search/validate/inspect responses exceeded the 10k inline threshold and
> spilled to HttpResponses files needing extra Reads."

Fully recoverable (read off disk), hence Low severity — but every Niagara author
building a stack pays a spill-to-file + extra Read on most of the ~14
`search_modules` discovery calls the build takes.

severity rationale: impact=response-spill (recovers via Read) x reach=niagara stack-authoring node-discovery (not every editor session) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the clean `niagara` "campfire embers reusable VFX from scratch" fuzz task (`/Game/FX/NS_CampfireEmbers`, outcome clean/done, judge filed nothing on the outcome). `niagara.search_modules` returns full per-hit multi-line descriptions + empty `inputs`/`outputs` arrays with no compact/projection mode, so most discovery queries overflowed the 10000-char inline threshold and spilled to `HttpResponses/<uuid>.json`, each forcing a `Grep '"(assetPath|name)":'` over the spill file to extract one path: `query="solve forces and velocity"` -> 59759 chars, `"color"` -> 22557, `"initialize"` limit15 -> 17875, `"spawn"` limit15 -> 16657; ~14 search_modules RPCs total, several spilling. Same no-compact-mode/verbose-per-hit spill family already tracked for `E-search-metasound-nodes-no-compact-mode` (MetaSound node discovery) and `E-console-search-default-limit-spills` (`system.console.search`), tracked per method; `niagara.search_modules` is the Niagara-module-discovery member with no such knob. Distinct from `E-niagara-standard-stack-recipe-undocumented` (same method but the orthogonal content/keyword-miss + missing-recipe gap, judge-filed for this same task) and `E-niagara-inspect-no-param-readback-projection` (different niagara method). Proposes `compact:true` (assetPath+name+usage+validStages, dropping the fat description + empty inputs/outputs) and/or a `fields`/`namesOnly` projection or a smaller default `limit`, keeping `totalMatches`/`truncated`; docs note on `docs/wiki-src/niagara.search_modules.md`. Low — recovers via disk Read but taxes most discovery searches of every Niagara stack build.
