---
id: E-get-graph-details-light-inventory-undiscoverable
title: "get_graph_details light node-inventory is the default but undiscoverable: callers reach for includeNodeDetails:true (spills) then guess namesOnly (rejected)"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, graph, response-size, projection, namesOnly, docs]
encounters: 3
lastSeen: 2026-06-29T02:33:49Z
---

# `blueprint.graph.get_graph_details` light inventory is the default, but nothing says so — callers overflow with `includeNodeDetails:true` then guess `namesOnly`

For the textbook "how many nodes / which entry nodes are in this graph" check,
`get_graph_details` **already** returns exactly the lightweight inventory the
caller wants — but only if they *omit* `includeNodeDetails`. The default (no
`includeNodeDetails`) shape emits a top-level `nodeCount` plus, per node, just
`{nodeId, nodeName, nodeTitle}`
(`Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphInspectionHandler.cpp:487`,
`:496-505`). Passing `includeNodeDetails:true` switches each node to the full
`BuildNodeDetailsJson` payload (every pin + adjacency) — the heavy shape that
overflows the inline budget.

Nothing in the registration/wiki signals this. The natural reading of the param
list (`includeNodeDetails`, `includePinDefaults`, `includeNodeState`,
`includeConnections` — all `include*` opt-ins) is "turn things on to see node
detail," so a caller who wants a node inventory reaches for
`includeNodeDetails:true`, gets the verbose payload, and overflows. Then, trying
to dial it back to a names-only view, they guess a `namesOnly` projection (the
convention `actor.list` / `system.inspect.list_objects` / `gameplay_tags.list` /
`skeleton.list_physics_bodies` / `system.console.search` all ship via the shared
`FHandlerContext::ReadFieldProjection`) — which `get_graph_details` rejects.

A second, subtler ergonomic snag compounds the guess: the two sibling
graph-inspection readers **disagree on unknown-param handling**.
`get_graph_details` validates strictly and rejects `namesOnly` with
`[UNKNOWN_PARAMS] … Valid parameters: [assetPath, …, includeNodeDetails,
includePinDefaults, includeNodeState, includeConnections]`, whereas
`get_nodes` registers only `assetPath`/`graphName`/`includePinDefaults`/
`includeNodeState` (`:377-382`) and **silently swallows** an unknown `namesOnly`,
returning its full node list unprojected. So the fallback `get_nodes {namesOnly}`
*looks* like it honored the projection but didn't — it just ignored the param and
returned everything. (Neither sibling actually has a `namesOnly` projection; one
errors, one no-ops.)

## What it should do

Primarily a **docs** fix on the `get_graph_details` section of the overlay
(`docs/wiki-src/blueprint.graph.md`, served as
`wiki-generated/blueprint.graph.get_graph_details.md`):
- State that the **default** shape (omit `includeNodeDetails`) already returns
  `nodeCount` + a light per-node `{nodeId, nodeName, nodeTitle}` inventory — the
  right call for "count nodes / list entry nodes."
- State that `includeNodeDetails:true` is the **heavy** full-pins shape that can
  exceed the inline budget and spill to a `HttpResponses/<uuid>.json` file, and
  should be used only when per-node pin detail is actually needed.
- Note there is **no `namesOnly`** on this method (it errors) and that the light
  default is the inventory equivalent.

Optionally (ergonomic, lower priority): accept a `namesOnly`/`names_only` alias
that maps to the light default shape, for symmetry with the list-style handlers,
so a caller's reflexive guess just works instead of erroring.

## Distinct from / related

- `E-get-nodes-pins-spill-no-projection` (OPEN, Medium) — the same
  projection-gap *family* but on the sibling `get_nodes`, which has **no** light
  shape at all (it unconditionally emits full pins+adjacency, so it spills even
  on a tiny graph). This ticket's distinctive angle is the opposite: on
  `get_graph_details` the light inventory **already exists as the default** — the
  gap is discoverability + the `includeNodeDetails`/`namesOnly` param ergonomics,
  not a missing lever. The two are worked together (the docs note here, the code
  lever there).
- `E-get-nodes-no-count-field` (OPEN) — `get_nodes` lacks a `nodeCount`;
  `get_graph_details` already has one. Orthogonal.
- `B-unknown-params-error-suggests-deleted-question-mark-suffix` (DONE) — the
  `UNKNOWN_PARAMS` *message wording*; this ticket is about the param-surface
  asymmetry the message exposes, not the wording.

## Evidence

From the struggle audit of a clean BPIR idempotent re-upsert task (focus
`blueprint.compile_bpir`, namespace `blueprint`, outcome **clean**, 21 calls —
all but one `ok`/non-error). The single `is_error` was the param guess. On a
**14-node** `EventGraph`:
- `blueprint.graph.get_graph_details {includeNodeDetails:true}` overflowed at
  **26128 chars** (threshold 10000), spilled to a `HttpResponses/<uuid>.json`
  file, and forced an extra `Read` of that file just to count/inventory nodes.
- `blueprint.graph.get_graph_details {graphName:"EventGraph", namesOnly:true}` →
  `[UNKNOWN_PARAMS] Unknown parameter(s) for 'blueprint.graph.get_graph_details':
  [namesOnly]. Valid parameters: [assetPath, requestedPath, path, name,
  blueprintPath, blueprint_path, blueprintCandidates, candidates, graphName,
  includeNodeDetails, includePinDefaults, includeNodeState, includeConnections].`
- Recovered by re-issuing the same args to `blueprint.graph.get_nodes`, which
  returned the node list (14 nodes; `namesOnly` silently ignored, not projected).

The CallAnalyzer flagged the same call (`proposed` type E, severity low,
"get_graph_details lacks the namesOnly compact projection that sibling get_nodes
accepts"); corrected here for accuracy — `get_nodes` does **not** project on
`namesOnly`, it silently ignores it, and `get_graph_details`'s default shape is
already the light inventory the caller wanted.

Severity **Low**: pure friction — a response spill that only forces a `Read`
plus one misuse-then-correct recovered in a single call; `get_graph_details` is a
secondary inspection verb (not the every-session first call like `get_nodes`), and
the "count nodes" intent is already served inline by the default `nodeCount`, so
no reach bump.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean BPIR idempotent re-upsert task (focus `blueprint.compile_bpir`, 21 calls, one `is_error`, outcome `clean`). The agent wanted a node inventory on a 14-node EventGraph: `get_graph_details {includeNodeDetails:true}` overflowed at 26128 chars and spilled to `HttpResponses/<uuid>.json` (extra Read), then `get_graph_details {namesOnly:true}` was rejected `[UNKNOWN_PARAMS]`, then it fell back to `get_nodes` (which silently ignored `namesOnly`). Verified in source (`BlueprintGraphInspectionHandler.cpp`): `get_graph_details` already returns `nodeCount` (`:487`) + a light per-node `{nodeId, nodeName, nodeTitle}` array by default (`:496-505`) — exactly the inventory wanted; `includeNodeDetails:true` switches to the heavy `BuildNodeDetailsJson` full-pins shape that overflows. The sibling `get_nodes` registers only `assetPath`/`graphName`/`includePinDefaults`/`includeNodeState` (`:377-382`) and silently swallows unknown params, so neither sibling truly has a `namesOnly` projection (one errors, one no-ops). Proposed: docs note on the `get_graph_details` section of `docs/wiki-src/blueprint.graph.md` (default = light `nodeCount`+`{nodeId,nodeName,nodeTitle}` inventory; `includeNodeDetails:true` = heavy/spilling; no `namesOnly`); optionally a `namesOnly` alias mapping to the light default for symmetry. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — no existing `get_graph_details` projection/discoverability ticket; `E-get-nodes-pins-spill-no-projection` (sibling `get_nodes`, missing lever — related, distinct), `E-get-nodes-no-count-field` (orthogonal), `B-unknown-params-error-suggests-deleted-question-mark-suffix` (DONE, message wording) all distinct.
- `#2-bpir-idempotency-guid-diff-corroboration` `OPEN` reporter — Second instance, from a struggle audit of a clean-process BPIR idempotent re-upsert task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `ergo`; 13 real `mcp__pinwright__call` RPCs, all `ok`/non-error, zero retries, no param-format guessing — the task's own functional finding, GUID churn, is the judge's `E-compile-bpir-idempotent-omits-guid-regen`, distinct from this readback friction) on a fresh Actor BP `/Game/BP_BpirUpsertIdem` (3 custom events + wired downstream nodes, auto-layout → a 14-node-then-17-node EventGraph). To diff run-1 vs run-2 node **positions and GUIDs** for the idempotency check the agent ran `blueprint.graph.get_graph_details {graphName:"EventGraph", includeNodeDetails:true}` **twice** (RUN1, RUN2), and **both** overflowed the 10000-char threshold — **35403 chars** then **35415 chars** (≈3.5× budget) — spilling to `HttpResponses/<uuid>.json` files the agent had to `Read` to do the diff. Same root cause as #1 (caller reaches for `includeNodeDetails:true` and overflows). **New angle:** here the intent was node positions+GUIDs (not the count/entry-node inventory of #1), so this ticket's light default `{nodeId,nodeName,nodeTitle}` would **not** have served it — there is no projection *anywhere* returning just `nodeId/x/y/guid` inline (the cross-cutting projection gap tracked on the sibling `E-get-nodes-pins-spill-no-projection`, whose proposed `fields:[nodeId,x,y]` lever would). **Discoverability corroboration:** the agent reasoned in THINK that "no `blueprint.graph.get_nodes` exists" after reading the `find_nodes`/`get_node_details`/`list_graphs`/`get_graph_details` docs, then used `get_graph_details {includeNodeDetails:true}` as the fallback — reinforcing this ticket's "the right lean readback is undiscoverable" thesis, and arguing the proposed docs note should also cross-link `get_nodes` from the graph-readback overview. The CallAnalyzer's `proposed` (type docs, severity low) framed the fix as "surface `get_nodes(namesOnly/fields)` as the default lean readback" — but that repeats the same false-projection premise #1 already corrected: re-verified the generated `wiki-generated/blueprint.graph.get_nodes.md` lists only `assetPath`/`graphName`/`includePinDefaults`/`includeNodeState` and the source `BlueprintGraphInspectionHandler.cpp` registers no `namesOnly`/`fields` for `get_nodes` (only `get_graph_connections`/`get_node_details_batch` take a `nodeIds` allow-list), so even had the agent found `get_nodes` it would have spilled too. Same proposed fix as #1 (docs note on the `get_graph_details` section of `docs/wiki-src/blueprint.graph.md`, optional `namesOnly` alias to the light default), plus the cross-link to `get_nodes`; no new fix. Severity stays Low (pure friction — two response spills forcing Reads).
- `#3-smallest-graph-overflow-corroboration` `OPEN` reporter — Third instance (cross-task aggregation), from a CallAnalyzer call-trace finding on a clean `blueprint.compile_bpir` idempotent re-upsert task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`; 18 real `mcp__pinwright__call` RPCs, **all** `ok`/non-error, zero retries, no param-format guessing; transcript `agent-a86b84bc6623b01d6.jsonl`) on a fresh Actor BP `/Game/BP_IdemReupsert` (BeginPlay→branch→`ApplyScore(float,int)` custom event reading/writing `Score`+`HitCount`). The agent ran `blueprint.graph.get_graph_details {graphName:"EventGraph", includeNodeDetails:true}` once for the node inventory; it overflowed at **18196 chars** (threshold 10000, ≈1.8×) and spilled to `HttpResponses/20260629T021430Z/<uuid>.json`, forcing an extra `Read` to recover the tool's own output. **New angle:** this is the **smallest graph yet** to overflow — a **12-node** EventGraph (vs #1's 14 nodes / 26128 chars and #2's 14–17 nodes / 35403–35415 chars) — reinforcing that `includeNodeDetails:true` blows the inline budget on even a trivially small graph, so the mode is self-defeating for its stated purpose (detailed node inspection). Unlike #1/#2 this agent did **not** make the `namesOnly` guess: it used the lighter lenses (`list_graphs`, `decompile`, `find_orphaned_nodes`, `find_nodes`) for the actual checks — all returned inline without overflow — so here the friction is purely the spill, the param-ergonomics snag isn't exercised. The CallAnalyzer flagged this same call (`proposed` type E, severity low: "get_graph_details includeNodeDetails reliably overflows the 10k display cap on even a small graph") and marked it `title_dup:"x"` (an explicit duplicate of this ticket). Its proposed remedies — (a) a more compact per-node detail shape, (b) a paged / field-projected detail response, or (c) a docs note steering callers to the light default / `find_nodes` — align with this ticket's existing fix. No new fix; severity stays Low.
