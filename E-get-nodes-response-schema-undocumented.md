---
id: E-get-nodes-response-schema-undocumented
title: "blueprint.graph.get_nodes wiki page documents only params, not the response field names, so callers guess (posX/title) and must parse live output to learn x/y/nodeTitle"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, graph, docs, response-shape]
encounters: 1
lastSeen: 2026-06-27T15:41:46Z
---

# `blueprint.graph.get_nodes` response field schema is undocumented — callers parse output to learn the key names

The generated wiki page for `blueprint.graph.get_nodes`
(`wiki-generated/blueprint.graph.get_nodes.md`) documents **only the
parameters** (`assetPath`, `graphName`, `includePinDefaults`,
`includeNodeState`) and the one-line summary "Get all nodes in a blueprint
graph". It says **nothing about the shape of the response** — there is no list
of the per-node fields the call returns. So a caller who wants to read a node's
position or display name has no documented way to learn the JSON keys and falls
back to guessing conventional names, then has to execute the call and parse the
live (often spilled-to-disk) output to discover the **actual** keys.

The concrete trap this task hit: a caller reasonably expected `posX`/`posY` for
position and `title` for the display name, but `get_nodes` actually returns
**`x`/`y`** and **`nodeTitle`** (alongside `nodeId`/`nodeName`/`nodeType`/
`comment`/`pins`). That mismatch cost an extra parse iteration on a response
that had already spilled to a `HttpResponses/<uuid>.json` file — so the caller
was re-Reading the spilled JSON specifically to reverse-engineer field names the
docs could have stated up front.

## What it should do

Add a short **response-shape** section to the `get_nodes` page (via the
`docs/wiki-src/blueprint.graph.md` overlay, which is the source for the
generated page) listing the per-node fields the handler actually emits —
`nodeId`, `nodeName`, `nodeType`, `nodeTitle`, `comment`, `x`, `y`, `pins[]`
(with `linkedTo`), plus the `includeNodeState`-gated `nodeState` block — so the
canonical first call of every BP-graph workflow can be written against a
documented schema instead of a parse-and-discover round trip. Naming the
position keys (`x`/`y`, **not** `posX`/`posY`) and the title key (`nodeTitle`,
**not** `title`) explicitly is the high-value part, since those are the names
callers most often guess wrong.

## Distinct from

- `E-get-nodes-pins-spill-no-projection` (OPEN, judge-filed for this same task) —
  that ticket is about response **size** (the unnarrowable pins+adjacency
  payload spilling to disk) and proposes a code fix (add a `fields`/`namesOnly`/
  `limit` lever). This ticket is orthogonal: even with a projection lever, the
  **field names themselves are undocumented**, so the discovery cost persists.
  A docs fix here; a code fix there.
- `E-node-comment-field-name-drift` (OPEN) — that ticket is a cross-method
  **drift** (the comment field is `comment` in `get_nodes` but `nodeComment` in
  `get_node_details`). This ticket is not a drift between two methods; it is that
  the `get_nodes` response shape is **not documented at all**, so a first-time
  caller guesses key names (`posX`/`title`) that don't exist. Different root
  cause (missing docs vs. inconsistent keys) and different fix (document the
  response vs. align the two keys).
- `E-get-nodes-no-count-field` (OPEN) — missing top-level `nodeCount`; a
  response-content gap, not a docs/naming gap. Orthogonal.

## Evidence

From the struggle audit of a clean BPIR auto-layout round-trip task (focus
`blueprint.compile_bpir`, namespace `blueprint`, outcome **ergo** — all 18 calls
`ok`/non-error, zero retries; the round-trip itself passed). The friction note,
verbatim: *"only minor note is get_nodes responses exceeded the 10k display
threshold and spilled to a file I had to parse, and node fields use x/y/nodeTitle
(not posX/title) which took one parse iteration to discover."* The call log shows
two `blueprint.graph.get_nodes {includeNodeState:true}` calls on
`/Game/BP_BpirAutoLayoutFuzz` (cross-check coords on v1, idempotency node-count
on v2); both spilled, and the field-name guess (`posX`/`title`) had to be
corrected to the actual `x`/`y`/`nodeTitle` by parsing the spilled JSON.
Confirmed in docs: `wiki-generated/blueprint.graph.get_nodes.md` lists only the
four parameters and carries no response-field documentation.

Severity Low: a pure docs/discoverability gap whose cost is a **one-time** parse
iteration to learn the key names (not a per-call tax), kept at Low rather than
bumped for `get_nodes` being an every-session method — the confusion bites once
per caller, then the names are known. Mirrors the Low of the analogous
same-method naming ticket `E-node-comment-field-name-drift`.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean BPIR auto-layout round-trip (focus `blueprint.compile_bpir`, namespace `blueprint`, 18 calls all `ok`/non-error, zero retries, outcome `ergo`). Friction note verbatim: *"node fields use x/y/nodeTitle (not posX/title) which took one parse iteration to discover."* The task made two `blueprint.graph.get_nodes {includeNodeState:true}` calls on `/Game/BP_BpirAutoLayoutFuzz`, both of which spilled to `HttpResponses/<uuid>.json`; the caller had guessed `posX`/`title` and had to parse the spilled JSON to find the real keys `x`/`y`/`nodeTitle`. Verified in docs: the generated page `wiki-generated/blueprint.graph.get_nodes.md` documents only the params (`assetPath`/`graphName`/`includePinDefaults`/`includeNodeState`) and has no response-shape section, so the per-node field names (`nodeId`/`nodeName`/`nodeType`/`nodeTitle`/`comment`/`x`/`y`/`pins`) are undiscoverable without executing the call. Proposed: add a response-shape section to the `docs/wiki-src/blueprint.graph.md` overlay listing those fields, naming `x`/`y` (not `posX`/`posY`) and `nodeTitle` (not `title`) explicitly. Dedup: ripgrep across OPEN/closed — `E-get-nodes-pins-spill-no-projection` (size/projection, code fix), `E-node-comment-field-name-drift` (cross-method drift), and `E-get-nodes-no-count-field` (missing count) are all distinct; no ticket documents the `get_nodes` response schema or the `x`/`y`/`nodeTitle` naming.
