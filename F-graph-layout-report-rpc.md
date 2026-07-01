---
id: F-graph-layout-report-rpc
title: "Runtime layout-quality report RPCs per graph domain"
status: OPEN
severity: Medium
category: feature
tags: [layout, metrics, rpc, blueprint, material, anim, controlrig]
blockedBy: [F-graph-layout-metrics-core]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# Runtime layout-quality report RPCs per graph domain

The layout metrics from `F-graph-layout-metrics-core` are useful at build
time, but there is no runtime surface for an agent (or the test-workflow's
layout-quality flagging) to measure an existing live graph. Expose the
metrics as read-only report RPCs, one per graph domain:

- `blueprint.graph.layout_report`
- `material.graph.layout_report`
- `anim.graph.layout_report`
- `controlrig.graph.layout_report`

**Niagara is excluded:** confirmed there is no Niagara layout engine — the
`Source/PinWright/Private/NIR/` directory contains `NIRDecompiler`,
`NIRGraphEmitter_*`, and `NIRTextEmitter` only, no `NIRLayoutEngine`. Without
a layout engine to measure against there is nothing to report, so no Niagara
variant is filed here. Add it later only if a Niagara layout engine lands.

## What each handler should do

For its graph domain:
1. Gather node positions via `Node->NodePosX` / `Node->NodePosY`.
2. Gather node sizes via the metrics-core node-size estimator
   (`F-graph-layout-metrics-core`) using the matching per-type adapter.
3. Gather connections by reusing the existing connection-iteration path —
   `blueprint.graph.get_graph_connections` already exists
   (`Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphInspectionHandler.cpp:524`).
4. Feed `[{nodeId,x,y,w,h}]` + edges into `FGraphLayoutMetrics`.
5. Return the sub-scores + combined 0-1 score, plus flagged problem nodes:
   overlapping nodes, over-long edges, and tangled (high-crossing) edges.

## Proposed approach

Follow the existing read-only inspection handler scaffolding in
`Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphInspectionHandler.cpp`:
`REGISTER_RPC_HANDLER` → `ResolveBlueprintAndGraph(Ctx, Blueprint, Graph)`
(line 386 pattern) → iterate `Graph->Nodes` (line 392) → build result →
`Ctx.SendSuccess(Result)`. Mirror that shape into the material / anim /
controlrig handler files, resolving each domain's graph the way that domain's
existing handlers already do.

This is the runtime surface the **test-workflow's layout-quality flagging**
will consume to catch poor layout in CI / verification runs.

## Severity justification

**Medium.** Soft blocker / missing readback verb: today an agent cannot
measure a live graph's layout at all and must eyeball it. No crash, no data
corruption — it is a read-only report. Blocked by
`F-graph-layout-metrics-core`, which supplies the estimator and
`FGraphLayoutMetrics`.

## History
- `#1-initial-spec` `OPEN` reporter — No runtime surface to measure a live graph's layout; add read-only layout_report RPCs for blueprint/material/anim/controlrig (Niagara excluded — no NIR layout engine) that gather NodePosX/Y + estimator sizes + get_graph_connections into FGraphLayoutMetrics and return scores plus flagged overlapping/long/tangled nodes, scaffolded like BlueprintGraphInspectionHandler.
