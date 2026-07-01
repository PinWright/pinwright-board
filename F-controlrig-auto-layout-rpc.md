---
id: F-controlrig-auto-layout-rpc
title: "Standalone controlrig.graph.auto_layout RPC (no compile round-trip)"
status: OPEN
severity: Medium
category: feature
tags: [layout, controlrig, crir, rpc, authoring]
blockedBy: [F-graph-layout-metrics-core]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# Standalone controlrig.graph.auto_layout RPC (no compile round-trip)

ControlRig-graph auto-layout is reachable today only as a side effect of CRIR
compile (`FCRIRLayoutEngine` runs inside the compile path). After an agent
builds a rig graph imperatively (per-node creation with guessed, copied, or
omitted x/y) there is no way to re-flow the existing graph without a full CRIR
compile round-trip.

This is the ControlRig parallel of the already-shipped
`material.authoring.auto_layout`
(`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3360`).
Note: material **already** has its standalone auto-layout RPC, so no material
variant is filed (and none should be).

## What it should do

Add a standalone auto-layout RPC (`controlrig.graph.auto_layout`, or the
appropriate ControlRig namespace) that re-flows a target rig graph via
`FCRIRLayoutEngine`
(`Source/PinWright/Private/CRIR/CRIRLayoutEngine.cpp`) **without** a compile
round-trip, preserving its existing pre-positioned-obstacle awareness. Lets an
agent re-flow and then re-measure (via `controlrig.graph.layout_report`,
`F-graph-layout-report-rpc`) after imperative node creation.

## Proposed approach

- Mirror `material.authoring.auto_layout`'s structure: resolve the target
  ControlRig + RigVM graph the way the existing ControlRig handlers do, run
  `FCRIRLayoutEngine::RunLayout` over it (keeping its pre-positioned-obstacle
  handling), then `MarkPackageDirty` — no compile, no save flag (caller
  invokes `editor.save_all` explicitly).
- Return a small result: resolved target, nodes laid out, durationMs (same
  shape family as `material.authoring.auto_layout`).
- Once `F-graph-layout-metrics-core` lands, the engine's measured sizing
  feeds the same estimator, so re-flow and report agree on bounds.

## Severity justification

**Medium.** Soft blocker: re-flowing an imperatively built rig graph is only
possible via a heavy CRIR compile round-trip today. No crash, no data
corruption. Blocked by `F-graph-layout-metrics-core` so the standalone
re-flow uses the shared node-size estimator and stays consistent with the
report RPC.

## History
- `#1-initial-spec` `OPEN` reporter — ControlRig auto-layout is only a CRIR-compile side effect; add a standalone controlrig.graph.auto_layout RPC that re-flows via FCRIRLayoutEngine (keeping pre-positioned-obstacle awareness) without a compile round-trip, mirroring the shipped material.authoring.auto_layout (MaterialAuthoringHandler.cpp:3360); material already has its own, so none is filed for material.
