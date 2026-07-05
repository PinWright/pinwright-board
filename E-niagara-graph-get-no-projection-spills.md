---
id: E-niagara-graph-get-no-projection-spills
title: "niagara.graph.get has no node/pin projection or single-node filter — even a 28-node graph serializes to ~207KB, spills to a file too large to Read, and forces external jq"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, niagara-graph, graph-get, response-size, oversized, projection, spills, docs]
encounters: 1
lastSeen: 2026-07-05T11:42:41.6664801+03:00
---

# `niagara.graph.get` has no compact/projection mode — a small script graph spills, and the spill is too big to Read

`niagara.graph.get` is the read call for a Niagara script graph (the first call
of every graph-edit workflow: see what you're working with, pick a node), but it
emits, **for every node**, every pin's full type object plus
`present`/`defaultValue`/`defaultObject`/`defaultText` fields, with **no
narrowing lever** — no `namesOnly`/`fields` projection that drops the heavy
per-pin default metadata, no single-node `nodeId` filter, and no companion
"list nodes" (ids + titles + pin names only). So the response size scales with
(nodes × pins × per-pin metadata) and the caller cannot ask for the lightweight
"what nodes exist / which one do I want" shape the read intent actually needs.

The practical effect is worse than the usual spill: the payload is so large that
even the spilled file is too big to `Read`, forcing a fall-back to external
`jq`:

- `niagara.graph.get` (Simple_system ParticleUpdate, **28 nodes**) →
  `outputTooLong` at **207308 chars** (threshold 10000), spilled to a JSON file.
- A direct `Read` of that spill file **also failed**:
  `File content (90372 tokens) exceeds maximum allowed tokens (25000).`
- Recovery required **~5 `jq` Bash calls** to extract the emitter name, node
  ids/titles, and pin types (one `jq` even errored first on an invalid regex
  escape).
- The post-save readback repeated the identical pattern: another spill at
  **214341 chars** + more `jq`.

So the one method whose job is "show me the graph" cannot stay inline, cannot be
recovered by a single `Read`, and turns every graph inspection into a
spill-file + external-`jq` detour.

## What it should do

Give `niagara.graph.get` a narrowing lever so the common "read to pick" call
stays inline (and, failing that, is recoverable in one `Read`), mirroring the
projection/limit levers the board has already shipped/proposed across the verbose
readers:

- A **`namesOnly`/`fields` projection** returning just node id / title / type
  (and optionally pin **names**) and **omitting** the heavy per-pin type object
  and `defaultValue`/`defaultObject`/`defaultText` metadata unless requested.
- And/or a **single-node `nodeId` filter** so a targeted readback (verify one
  just-created node's pins) returns only that node inline.
- And/or a companion **`niagara.graph.list_nodes`** (ids + titles + pin names
  only) so enumerate-to-pick never drags full pin defaults.
- **Docs (`docs/wiki-src/niagara.graph.md`):** note that a full `graph.get` of a
  real script graph exceeds the inline budget and spills (and that the spill file
  can itself exceed the `Read` cap), and that the projection / single-node filter
  keeps a read inline — so the overflow is a documented expectation.

## Distinct from / related

- `E-niagara-inspect-no-param-readback-projection` (IN-REVIEW) — same
  response-size **family** but a **different niagara method** (`niagara.inspect`,
  the whole-asset structural dump). Its `#5`/`#6` history even records callers
  falling back to `niagara.graph.get` to hand-trace the ParameterMap chain — i.e.
  `graph.get` is the method they escape *to*, and it has the same spill problem,
  which this ticket tracks for that method.
- `E-niagara-search-modules-no-compact-spills` (OPEN) — same no-compact/verbose
  spill shape on the Niagara **module-discovery search**, not the script-graph
  read. Board tracks the family per method; this is the `niagara.graph.get`
  member.
- `E-get-nodes-pins-spill-no-projection` (IN-REVIEW) — the exact same
  "graph reader emits full per-node pins+adjacency, no `fields`/`namesOnly`/
  `limit`, so even a tiny graph spills" shape on the **Blueprint** side
  (`blueprint.graph.get_nodes`). Same proposed fix family, different graph
  domain; none of its history names `niagara.graph.get`.
- `E-http-response-spill` (DONE) — the generic server-side spill mechanism; this
  ticket is a specific verbose reader overflowing by default (with the added
  wrinkle that the spill file itself exceeds the `Read` token cap).

## Evidence

PROCESS friction from the clean/`done` `niagara.graph.create_node` task on
`/Game/ExampleContent/Niagara/Simple/Simple_system` (emitter `Simple_Emitter`,
ParticleUpdate; 10 MCP RPCs, the focus `create_node` itself clean). The task's
two `niagara.graph.get` calls (initial 28-node read + post-save 29-node readback)
both overflowed and both required `jq`.

CallAnalyzer call-trace finding, verbatim:

> "niagara.graph.get on Simple_system ParticleUpdate returned outputTooLong:
> 207308 chars (threshold 10000), spilled to a JSON file. A direct Read of that
> file failed: 'File content (90372 tokens) exceeds maximum allowed tokens
> (25000).' The attempt then ran ~5 jq Bash calls to extract the emitter name,
> node ids/titles, and pin types (one jq even errored first on an invalid regex
> escape). The post-save readback repeated the same pattern: another 214341-char
> spill + jq. Each of the 28 nodes carried every pin's full type object plus
> present/defaultValue/defaultObject/defaultText fields."

Agent friction note, verbatim:

> "Large graph.get responses spilled to files, needing jq to inspect."

## Severity rationale

severity rationale: impact=response-spill that forces a Read (here worse — the
spill file exceeds the Read cap, forcing external `jq`) × reach=rare
(raw Niagara script-graph reads, not an every-session path) → Low (spill floor
per the rubric; no upward reach bump because the raw-graph read path is
infrequent — the spill-file-too-big-for-Read wrinkle raises the tax but not the
severity class, since the data is still fully recoverable off disk).

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the clean/`done` `niagara.graph.create_node` task on `/Game/ExampleContent/Niagara/Simple/Simple_system` (emitter `Simple_Emitter`, ParticleUpdate; 10 MCP RPCs; the focus `create_node` worked first-try and the judge filed nothing on the outcome). `niagara.graph.get` has no `namesOnly`/`fields` projection, no single-node `nodeId` filter, and no compact `list_nodes` companion, so it emits every node's full per-pin type object + `present`/`defaultValue`/`defaultObject`/`defaultText`. On this **28-node** graph it spilled at **207308 chars** (post-save readback **214341 chars**), and the spill file was itself too large to `Read` (`90372 tokens > 25000 cap`), forcing ~5 `jq` Bash calls (one erroring first on a bad regex escape) to pull out emitter name / node ids/titles / pin types. Proposes a `namesOnly`/`fields` projection (drop the per-pin default metadata), a single-node `nodeId` filter, and/or a `niagara.graph.list_nodes` companion, plus a `docs/wiki-src/niagara.graph.md` note that a full `graph.get` spills (and the spill can exceed the Read cap). Dedup: ripgrep across OPEN/closed found no ticket naming `niagara.graph.get` as a spilling reader; `E-niagara-inspect-no-param-readback-projection` (different method — inspect; its history escapes *to* graph.get), `E-niagara-search-modules-no-compact-spills` (module search, not graph read), and `E-get-nodes-pins-spill-no-projection` (Blueprint graph reader) are the same family on different methods, all distinct.
