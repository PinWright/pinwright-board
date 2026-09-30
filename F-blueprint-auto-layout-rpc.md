---
id: F-blueprint-auto-layout-rpc
title: "Standalone blueprint.graph.auto_layout RPC (no compile round-trip)"
status: OPEN
severity: Low
category: feature
tags: [layout, blueprint, bpir, rpc, authoring, gap-analysis-2026-09-30]
blockedBy: [F-graph-layout-metrics-core, F-graph-layout-core]
encounters: 2
lastSeen: 2026-07-04T12:00:00Z
---

# Standalone blueprint.graph.auto_layout RPC (no compile round-trip)

Blueprint-graph auto-layout is reachable today only as a side effect of BPIR
compile (`compile_bpir` / `insert_bpir_at_node` run `FNodeLayoutEngine`). After
an agent builds a graph imperatively (per-node creation with guessed,
copied, or omitted x/y) there is no way to ask the editor to re-flow the
existing graph without paying for a full BPIR compile round-trip.

This is the Blueprint parallel of the already-shipped
`material.authoring.auto_layout`
(`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3360`),
which lifts the material layout helper out of the MGIR compile side-effect
flag.

## What it should do

Add a standalone `blueprint.graph.auto_layout` RPC that re-flows a target
Blueprint graph via `FNodeLayoutEngine`
(`Source/PinWright/Private/Compiler/NodeLayoutEngine.cpp`) **without** a
compile round-trip. Lets an agent re-flow and then re-measure (via
`blueprint.graph.layout_report`, `F-graph-layout-report-rpc`) after
imperative node creation.

## Proposed approach

- Mirror `material.authoring.auto_layout`'s structure: resolve the target
  Blueprint + graph (reuse `ResolveBlueprintAndGraph` from
  `BlueprintGraphInspectionHandler.cpp`), run `FNodeLayoutEngine` over
  `Graph->Nodes`, then `MarkPackageDirty` — no compile, no save flag (caller
  invokes `editor.save_all` explicitly).
- Return a small result: resolved target, nodes laid out, durationMs (same
  shape family as `material.authoring.auto_layout`).
- Once `F-graph-layout-metrics-core` lands, the engine's measured sizing
  feeds the same estimator, so the re-flow and the report agree on bounds.

## API contract (gap analysis 2026-09-30, supersedes the approach above)

This verb must run the new layout core, `F-graph-layout-core`, not the current
`FNodeLayoutEngine`, which is being replaced (`B-layout-engine-ba-derived`).
`material.authoring.auto_layout`, `anim.graph.auto_layout` and
`controlrig.graph.auto_layout` share this exact contract (rpc-design §21).

**Request:** `blueprint.graph.auto_layout {assetPath, graphName, scope, nodes?, anchor?, dryRun?}`

- `scope` — **required, no default** (rpc-design §3). One of:
  - `"all"`: every node in the graph.
  - `"nodes"`: the ids in `nodes[]`; required and non-empty in this mode.
  - `"selection"`: the current graph-editor selection. Error when no editor is open.
  - `"unpositioned"`: nodes still at (0,0).

  Nodes outside the scope are fixed obstacles. An empty resolved scope is an error,
  not a zero-item success.
- `anchor` — optional node id that keeps its position. By default each root keeps
  its position.
- `dryRun` — compute and report without writing.

**Response:**

- `moved[{nodeId, title, from:{x,y}, to:{x,y}}]`, `movedCount`, `unchangedCount`.
  Positions are read back from the nodes after the write, never echoed from the plan.
- `sizeSource {measured, estimated}` — how many node sizes were measured
  (live or offscreen panel) versus estimated (`F-graph-node-size-measured`). Never
  report an estimate as measured.
- `metrics {before, after}`, each `{overlaps, backwardEdges, crossings, score}`
  from `GraphLayoutMetrics` (`E-layout-metrics-backward-and-pin-crossings`). The
  `after` values are re-measured from the read-back positions, using measured sizes
  where available.
- `transaction` — the name of the single `FScopedTransaction` wrapping the call.
  Moves go through `Schema->SetNodePosition` (which calls `Modify()`). The
  transaction is cancelled when `movedCount` is 0. Undo restores every position.
- `commentsRefit[]` — once `F-graph-layout-comment-fit` lands.

**Acceptance:**
- Missing `scope` fails the required-parameter gate. An empty `nodes[]` errors.
- On an already laid-out graph: `movedCount: 0` and no transaction.
- On a messy fixture: `metrics.after.overlaps == 0` and
  `metrics.after.backwardEdges == 0`, excluding cycle back edges.
- Undo restores every `from` position.
- Determinism: two runs on the same input give identical `to` positions.
- Vision check: the graph is captured with `editor.frame_graph` +
  `editor.screenshot_window` and looked at (rpc-design §16).

## Severity justification

**Medium.** Soft blocker: re-flowing an imperatively built graph is only
possible via a heavy BPIR compile round-trip today. No crash, no data
corruption. Blocked by `F-graph-layout-metrics-core` so the standalone
re-flow uses the shared node-size estimator and stays consistent with the
report RPC.

## History
- `#1-initial-spec` `OPEN` reporter — Blueprint auto-layout is only a BPIR-compile side effect; add a standalone blueprint.graph.auto_layout RPC that re-flows via FNodeLayoutEngine without a compile round-trip, mirroring the shipped material.authoring.auto_layout (MaterialAuthoringHandler.cpp:3360), so agents can re-flow + re-measure after imperative node creation.
- `#2-crossing-wire-repro` `OPEN` reporter — additional-evidence: rebuilt BP_ScoreBus AddPoints via blueprint.compile_bpir (5 nodes: AddPoints → Add_IntInt → Set Score → CallOnScoreChanged + Get Score); the Get-Score node's data wires came out crossing under other nodes and there was no way to re-flow the compiled graph, confirming this gap on a real screenshot/docs use-case. Grep of REGISTER_RPC_HANDLER across ...\Handlers\ found no blueprint.* auto-layout/arrange RPC (material.authoring.auto_layout at MaterialAuthoringHandler.cpp:3379 remains the only graph auto-layout verb). Correction to the incoming report's second half: a per-node reposition RPC ALREADY exists — blueprint.graph.set_node_property with propertyName "X"/"Y" writes NodePosX/NodePosY (BlueprintGraphCrudHandler.cpp:2006-2019) — so an agent CAN hand-tidy node-by-node today; the genuine remaining gap is this ticket's bulk standalone re-flow, not a node-move verb. Note: re-flow via the same FNodeLayoutEngine won't by itself remove crossings — edge-crossing reduction / layout quality is tracked separately by the F-graph-layout-metrics-core cluster (edge-crossings metric + downstream edge-crossing follow-up), not here. Severity unchanged (Medium): soft blocker, workaround exists.
- `#3-api-contract-and-core-dependency` `OPEN` reporter — Gap analysis 2026-09-30: added the shared auto-layout API contract (required `scope` all|nodes|selection|unpositioned, `moved[]` from read-back positions, `sizeSource {measured, estimated}`, `metrics {before, after}` incl. backwardEdges, single cancellable transaction, determinism + undo acceptance). Now blocked on F-graph-layout-core: the verb must run the new layout core, not the FNodeLayoutEngine being replaced under B-layout-engine-ba-derived.
- `#4-re-rated` `OPEN` triage — Severity Medium -> Low. Graph layout is cosmetic (Low impact class); re-flow is reachable via a BPIR compile and per-node `blueprint.graph.set_node_property X/Y` exists (`#2`), and a standalone re-flow is not an every-session need, so no reach bump.
