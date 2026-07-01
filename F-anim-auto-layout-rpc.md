---
id: F-anim-auto-layout-rpc
title: "Standalone anim.graph.auto_layout RPC (no compile round-trip)"
status: OPEN
severity: Medium
category: feature
tags: [layout, anim, agir, rpc, authoring]
blockedBy: [F-graph-layout-metrics-core]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# Standalone anim.graph.auto_layout RPC (no compile round-trip)

Anim-graph auto-layout is reachable today only as a side effect of AGIR
compile (`FAGIRLayoutEngine` runs inside the compile path). After an agent
builds an anim graph imperatively (per-node creation with guessed, copied, or
omitted x/y) there is no way to re-flow the existing graph without a full AGIR
compile round-trip.

This is the anim parallel of the already-shipped
`material.authoring.auto_layout`
(`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3360`).

## What it should do

Add a standalone auto-layout RPC (`anim.graph.auto_layout`, or the
appropriate anim namespace) that re-flows a target anim graph via
`FAGIRLayoutEngine`
(`Source/PinWright/Private/AGIR/AGIRLayoutEngine.cpp`) **without** a compile
round-trip, including its recursive state-machine inner graphs. Lets an agent
re-flow and then re-measure (via `anim.graph.layout_report`,
`F-graph-layout-report-rpc`) after imperative node creation.

## Proposed approach

- Mirror `material.authoring.auto_layout`'s structure: resolve the target
  anim Blueprint + graph the way the existing anim handlers do, run
  `FAGIRLayoutEngine::Layout` over it (the engine already recurses into
  state-machine inner graphs), then `MarkPackageDirty` — no compile, no save
  flag (caller invokes `editor.save_all` explicitly).
- Return a small result: resolved target, nodes laid out, durationMs (same
  shape family as `material.authoring.auto_layout`).
- Once `F-graph-layout-metrics-core` lands, the engine's measured sizing
  feeds the same estimator, so re-flow and report agree on bounds.

## Severity justification

**Medium.** Soft blocker: re-flowing an imperatively built anim graph is only
possible via a heavy AGIR compile round-trip today. No crash, no data
corruption. Blocked by `F-graph-layout-metrics-core` so the standalone
re-flow uses the shared node-size estimator and stays consistent with the
report RPC.

## History
- `#1-initial-spec` `OPEN` reporter — Anim auto-layout is only an AGIR-compile side effect; add a standalone anim.graph.auto_layout RPC that re-flows via FAGIRLayoutEngine (including state-machine inner graphs) without a compile round-trip, mirroring the shipped material.authoring.auto_layout (MaterialAuthoringHandler.cpp:3360), so agents can re-flow + re-measure after imperative node creation.
