---
id: E-get-graph-details-light-inventory-undiscoverable
title: "get_graph_details: light node-inventory is the default but undiscoverable, and it is the lone graph reader that rejects namesOnly/fields (both siblings now accept them)"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [blueprint, graph, response-size, projection, namesOnly, docs]
encounters: 3
lastSeen: 2026-06-29T02:33:49Z
---

# `blueprint.graph.get_graph_details` light inventory is the default, but nothing says so — and it is the lone graph reader that rejects `namesOnly`/`fields`

For the textbook "how many nodes / which entry nodes are in this graph" check,
`get_graph_details` **already** returns exactly the lightweight inventory the
caller wants — but only if they *omit* `includeNodeDetails`. The default (no
`includeNodeDetails`) shape emits a top-level `nodeCount`
(`Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphInspectionHandler.cpp:554`)
plus, per node, just `{nodeId, nodeName, nodeTitle}` (`:563-572`). Passing
`includeNodeDetails:true` switches each node to the full `BuildNodeDetailsJson`
payload (every pin + adjacency) — the heavy shape that overflows the inline
budget on even a ~12-node graph.

Nothing in the registration/wiki signals this. The natural reading of the param
list (`includeNodeDetails`, `includePinDefaults`, `includeNodeState`,
`includeConnections` — all `include*` opt-ins) is "turn things on to see node
detail," so a caller who wants a node inventory reaches for
`includeNodeDetails:true`, gets the verbose payload, and overflows. Then, trying
to dial it back to a names-only view, they guess a `namesOnly` projection — which
`get_graph_details` rejects with `[UNKNOWN_PARAMS]` (strict validation in
`Dispatch/RpcDispatcher.cpp:127-139`; the registration at `:532-541` lists no
`namesOnly`/`fields`).

That guess is well-founded, and the asymmetry is now the sharp edge: **both**
sibling graph-inspection readers ship the shared `namesOnly`/`fields` projection
lever (parsed by `FHandlerContext::ReadFieldProjection`) — `get_nodes` registers
`fields`/`namesOnly`/`limit` and honors them (`:405-408`, applied at `:423`), and
`get_node_details_batch` registers `fields`/`namesOnly` (`:725-726`, applied at
`:748`). Those levers landed in commits `9aed16c` / `3609d86` (both ancestors of
HEAD; the sibling ticket `E-get-nodes-pins-spill-no-projection`). So
`get_graph_details` is the **lone holdout** in the graph-inspection family: the one
reader that both hides its light default and errors on the `namesOnly` a caller
reflexively reaches for after seeing its siblings accept it.

> Note (corrected): earlier history entries state that `get_nodes` "silently
> swallows" `namesOnly` and that "neither sibling has a `namesOnly` projection
> (one errors, one no-ops)." That was true at filing but is now **false** — the
> sibling projection lever has since landed (`9aed16c`/`3609d86`), inverting the
> asymmetry: the siblings accept `namesOnly`, and `get_graph_details` alone rejects it.

## What it should do

Primary — **code lever** (promote the alias to the primary fix now that both
siblings established the pattern in the same file): add `namesOnly`/`names_only`
and `fields` to `get_graph_details`, parsed by the shared
`FHandlerContext::ReadFieldProjection` and applied through the shared
`BuildNodeDetailsJson` field filter, exactly mirroring `get_nodes` (`:423`) and
`get_node_details_batch` (`:748`). `namesOnly` returns the light identification
set (`nodeId/nodeName/nodeType/nodeTitle/x/y`, pins dropped); `fields` is an
explicit per-key allow-list. Both **take precedence over `includeNodeDetails`**,
so a caller who overflowed with `includeNodeDetails:true` can add `namesOnly:true`
to the same args to dial back to an inline shape. This makes the reflexive guess
just work and makes the whole graph-inspection family uniform.

Secondary — **docs** note on the `get_graph_details` section of the overlay
(`docs/wiki-src/blueprint.graph.md`, served as
`wiki-generated/blueprint.graph.get_graph_details.md`): state that the default
(omit `includeNodeDetails`) is already the light `nodeCount` +
`{nodeId, nodeName, nodeTitle}` inventory; that `includeNodeDetails:true` is the
heavy full-pins shape that can spill; and that `namesOnly`/`fields` give a lean
projected read that overrides `includeNodeDetails`.

## Distinct from / related

- `E-get-nodes-pins-spill-no-projection` (IN-REVIEW) — the same projection-gap
  *family* on the sibling `get_nodes`; its `fields`/`namesOnly`/`limit` lever has
  since landed (commit `9aed16c`), which is what inverted this ticket's premise.
  This ticket brings the **same** lever to `get_graph_details` (the lone remaining
  holdout) plus the discoverability docs note for its already-existing light default.
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
accepts"). At the time of this encounter `get_nodes` did not yet project on
`namesOnly` (it silently ignored it) and `get_graph_details`'s default shape was
already the light inventory the caller wanted. **Since then the sibling projection
lever landed** (`9aed16c`/`3609d86`), so the CallAnalyzer's original framing is now
the correct one: `get_nodes` *does* accept `namesOnly`, and `get_graph_details` is
the lone reader still missing it — which this ticket now adds.

Severity **Low**: pure friction — a response spill that only forces a `Read`
plus one misuse-then-correct recovered in a single call; `get_graph_details` is a
secondary inspection verb (not the every-session first call like `get_nodes`), and
the "count nodes" intent is already served inline by the default `nodeCount`, so
no reach bump.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean BPIR idempotent re-upsert task (focus `blueprint.compile_bpir`, 21 calls, one `is_error`, outcome `clean`). The agent wanted a node inventory on a 14-node EventGraph: `get_graph_details {includeNodeDetails:true}` overflowed at 26128 chars and spilled to `HttpResponses/<uuid>.json` (extra Read), then `get_graph_details {namesOnly:true}` was rejected `[UNKNOWN_PARAMS]`, then it fell back to `get_nodes` (which silently ignored `namesOnly`). Verified in source (`BlueprintGraphInspectionHandler.cpp`): `get_graph_details` already returns `nodeCount` (`:487`) + a light per-node `{nodeId, nodeName, nodeTitle}` array by default (`:496-505`) — exactly the inventory wanted; `includeNodeDetails:true` switches to the heavy `BuildNodeDetailsJson` full-pins shape that overflows. The sibling `get_nodes` registers only `assetPath`/`graphName`/`includePinDefaults`/`includeNodeState` (`:377-382`) and silently swallows unknown params, so neither sibling truly has a `namesOnly` projection (one errors, one no-ops). Proposed: docs note on the `get_graph_details` section of `docs/wiki-src/blueprint.graph.md` (default = light `nodeCount`+`{nodeId,nodeName,nodeTitle}` inventory; `includeNodeDetails:true` = heavy/spilling; no `namesOnly`); optionally a `namesOnly` alias mapping to the light default for symmetry. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — no existing `get_graph_details` projection/discoverability ticket; `E-get-nodes-pins-spill-no-projection` (sibling `get_nodes`, missing lever — related, distinct), `E-get-nodes-no-count-field` (orthogonal), `B-unknown-params-error-suggests-deleted-question-mark-suffix` (DONE, message wording) all distinct.
- `#2-bpir-idempotency-guid-diff-corroboration` `OPEN` reporter — Second instance, from a struggle audit of a clean-process BPIR idempotent re-upsert task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `ergo`; 13 real `mcp__pinwright__call` RPCs, all `ok`/non-error, zero retries, no param-format guessing — the task's own functional finding, GUID churn, is the judge's `E-compile-bpir-idempotent-omits-guid-regen`, distinct from this readback friction) on a fresh Actor BP `/Game/BP_BpirUpsertIdem` (3 custom events + wired downstream nodes, auto-layout → a 14-node-then-17-node EventGraph). To diff run-1 vs run-2 node **positions and GUIDs** for the idempotency check the agent ran `blueprint.graph.get_graph_details {graphName:"EventGraph", includeNodeDetails:true}` **twice** (RUN1, RUN2), and **both** overflowed the 10000-char threshold — **35403 chars** then **35415 chars** (≈3.5× budget) — spilling to `HttpResponses/<uuid>.json` files the agent had to `Read` to do the diff. Same root cause as #1 (caller reaches for `includeNodeDetails:true` and overflows). **New angle:** here the intent was node positions+GUIDs (not the count/entry-node inventory of #1), so this ticket's light default `{nodeId,nodeName,nodeTitle}` would **not** have served it — there is no projection *anywhere* returning just `nodeId/x/y/guid` inline (the cross-cutting projection gap tracked on the sibling `E-get-nodes-pins-spill-no-projection`, whose proposed `fields:[nodeId,x,y]` lever would). **Discoverability corroboration:** the agent reasoned in THINK that "no `blueprint.graph.get_nodes` exists" after reading the `find_nodes`/`get_node_details`/`list_graphs`/`get_graph_details` docs, then used `get_graph_details {includeNodeDetails:true}` as the fallback — reinforcing this ticket's "the right lean readback is undiscoverable" thesis, and arguing the proposed docs note should also cross-link `get_nodes` from the graph-readback overview. The CallAnalyzer's `proposed` (type docs, severity low) framed the fix as "surface `get_nodes(namesOnly/fields)` as the default lean readback" — but that repeats the same false-projection premise #1 already corrected: re-verified the generated `wiki-generated/blueprint.graph.get_nodes.md` lists only `assetPath`/`graphName`/`includePinDefaults`/`includeNodeState` and the source `BlueprintGraphInspectionHandler.cpp` registers no `namesOnly`/`fields` for `get_nodes` (only `get_graph_connections`/`get_node_details_batch` take a `nodeIds` allow-list), so even had the agent found `get_nodes` it would have spilled too. Same proposed fix as #1 (docs note on the `get_graph_details` section of `docs/wiki-src/blueprint.graph.md`, optional `namesOnly` alias to the light default), plus the cross-link to `get_nodes`; no new fix. Severity stays Low (pure friction — two response spills forcing Reads).
- `#3-smallest-graph-overflow-corroboration` `OPEN` reporter — Third instance (cross-task aggregation), from a CallAnalyzer call-trace finding on a clean `blueprint.compile_bpir` idempotent re-upsert task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`; 18 real `mcp__pinwright__call` RPCs, **all** `ok`/non-error, zero retries, no param-format guessing; transcript `agent-a86b84bc6623b01d6.jsonl`) on a fresh Actor BP `/Game/BP_IdemReupsert` (BeginPlay→branch→`ApplyScore(float,int)` custom event reading/writing `Score`+`HitCount`). The agent ran `blueprint.graph.get_graph_details {graphName:"EventGraph", includeNodeDetails:true}` once for the node inventory; it overflowed at **18196 chars** (threshold 10000, ≈1.8×) and spilled to `HttpResponses/20260629T021430Z/<uuid>.json`, forcing an extra `Read` to recover the tool's own output. **New angle:** this is the **smallest graph yet** to overflow — a **12-node** EventGraph (vs #1's 14 nodes / 26128 chars and #2's 14–17 nodes / 35403–35415 chars) — reinforcing that `includeNodeDetails:true` blows the inline budget on even a trivially small graph, so the mode is self-defeating for its stated purpose (detailed node inspection). Unlike #1/#2 this agent did **not** make the `namesOnly` guess: it used the lighter lenses (`list_graphs`, `decompile`, `find_orphaned_nodes`, `find_nodes`) for the actual checks — all returned inline without overflow — so here the friction is purely the spill, the param-ergonomics snag isn't exercised. The CallAnalyzer flagged this same call (`proposed` type E, severity low: "get_graph_details includeNodeDetails reliably overflows the 10k display cap on even a small graph") and marked it `title_dup:"x"` (an explicit duplicate of this ticket). Its proposed remedies — (a) a more compact per-node detail shape, (b) a paged / field-projected detail response, or (c) a docs note steering callers to the light default / `find_nodes` — align with this ticket's existing fix. No new fix; severity stays Low.
- `#4-reword-and-implement` `IN-REVIEW` developer — Verified all three lens claims against plugin HEAD. CONFIRMED present: get_graph_details light default (`BlueprintGraphInspectionHandler.cpp:554` top-level `nodeCount` + `:563-572` per-node `{nodeId,nodeName,nodeTitle}`); registration (`:532-541`) carried no `namesOnly`/`fields`, so the dispatcher rejected them `[UNKNOWN_PARAMS]` (`RpcDispatcher.cpp:127-139`); overlay `Docs/wiki-src/blueprint.graph.md` had no `get_graph_details` section. STALE / inverted (the REWORD basis): the ticket's "second snag" — `get_nodes` silently swallows `namesOnly`, "neither sibling has a projection (one errors, one no-ops)" — is now FALSE; both siblings honor `namesOnly`/`fields` via `ReadFieldProjection` (`get_nodes` `:405-408` applied `:423`; `get_node_details_batch` `:725-726` applied `:748`; commits `9aed16c`/`3609d86`, both ancestors of HEAD), leaving get_graph_details the lone holdout. Reworded title/body/Fix to match (dropped the stale paragraph + added a corrected-note callout, refreshed the stale citations `:487`/`:496-505`/`:377-382` → `:554`/`:563-572`/`:398-408`/`:718-726`, reframed the alias as the PRIMARY code lever). Implemented that code lever: added `fields` + `namesOnly`/`names_only` to `get_graph_details`, parsed by `FHandlerContext::ReadFieldProjection` and applied through the shared `BuildNodeDetailsJson` field filter, taking precedence over `includeNodeDetails` so an overflowed `includeNodeDetails:true` call can add `namesOnly` to dial back inline; `namesOnly` → `nodeId/nodeName/nodeType/nodeTitle/x/y` (pins dropped). Added the secondary `### blueprint.graph.get_graph_details` docs section (default = light inventory; `includeNodeDetails:true` = heavy/spills; `namesOnly`/`fields` = lean projected read overriding `includeNodeDetails`). Files: `Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphInspectionHandler.cpp`, `Docs/wiki-src/blueprint.graph.md`. Regression test `PinWright.blueprint.graph.get_graph_details.FieldProjection` (`Source/PinWright/Private/Tests/Blueprint/TestGetGraphDetailsProjection.cpp`) asserts fields/namesOnly are registered, the default light shape is unchanged (no x, no pins), `includeNodeDetails:true` still emits pins, `namesOnly` adds nodeType/x/y and drops pins, `fields:["nodeId","x"]` keeps x / drops nodeTitle+pins, and `namesOnly` overrides `includeNodeDetails` (drops pins). Counterfactual: reverting drops the param registration and the projection, so the x/y/nodeType-present and pins-dropped assertions flip to failure.
