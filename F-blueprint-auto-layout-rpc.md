---
id: F-blueprint-auto-layout-rpc
title: "Standalone blueprint.graph.auto_layout RPC (no compile round-trip)"
status: OPEN
severity: Medium
category: feature
tags: [layout, blueprint, bpir, rpc, authoring]
blockedBy: [F-graph-layout-metrics-core]
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

## Severity justification

**Medium.** Soft blocker: re-flowing an imperatively built graph is only
possible via a heavy BPIR compile round-trip today. No crash, no data
corruption. Blocked by `F-graph-layout-metrics-core` so the standalone
re-flow uses the shared node-size estimator and stays consistent with the
report RPC.

## History
- `#1-initial-spec` `OPEN` reporter — Blueprint auto-layout is only a BPIR-compile side effect; add a standalone blueprint.graph.auto_layout RPC that re-flows via FNodeLayoutEngine without a compile round-trip, mirroring the shipped material.authoring.auto_layout (MaterialAuthoringHandler.cpp:3360), so agents can re-flow + re-measure after imperative node creation.
- `#2-crossing-wire-repro` `OPEN` reporter — additional-evidence: rebuilt BP_ScoreBus AddPoints via blueprint.compile_bpir (5 nodes: AddPoints → Add_IntInt → Set Score → CallOnScoreChanged + Get Score); the Get-Score node's data wires came out crossing under other nodes and there was no way to re-flow the compiled graph, confirming this gap on a real screenshot/docs use-case. Grep of REGISTER_RPC_HANDLER across ...\Handlers\ found no blueprint.* auto-layout/arrange RPC (material.authoring.auto_layout at MaterialAuthoringHandler.cpp:3379 remains the only graph auto-layout verb). Correction to the incoming report's second half: a per-node reposition RPC ALREADY exists — blueprint.graph.set_node_property with propertyName "X"/"Y" writes NodePosX/NodePosY (BlueprintGraphCrudHandler.cpp:2006-2019) — so an agent CAN hand-tidy node-by-node today; the genuine remaining gap is this ticket's bulk standalone re-flow, not a node-move verb. Note: re-flow via the same FNodeLayoutEngine won't by itself remove crossings — edge-crossing reduction / layout quality is tracked separately by the F-graph-layout-metrics-core cluster (edge-crossings metric + downstream edge-crossing follow-up), not here. Severity unchanged (Medium): soft blocker, workaround exists.
