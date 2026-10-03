---
id: F-anim-auto-layout-rpc
title: "Standalone anim.auto_layout RPC: re-flow positioned anim-graph nodes with the PwGraphLayout EdGraph adapter (AGIR compile only lays out nodes at (0,0))"
status: OPEN
severity: Low
category: feature
tags: [layout, anim, agir, rpc, authoring]
blockedBy: [F-graph-layout-metrics-core]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
rice: [1, 1, 0.8, 2]
priority: 3
---

# Standalone anim.auto_layout RPC (re-flow an existing anim graph)

There is no RPC that re-flows an anim blueprint's graphs. The only layout run is inside
`anim.compile_agir` (`runLayout`, default true;
`Source/PinWright/Private/Handlers/Animation/AGIRCompileHandler.cpp:60`), which calls
`PwGraphLayout::ArrangeAnimBlueprint` (`Source/PinWright/Private/AGIR/AGIRCompiler.cpp:1747-1750`).
That function moves only nodes still at (0,0) and treats every other node as a fixed obstacle
(`Source/PinWright/Private/Layout/PwGraphLayoutEdGraph.cpp:235-265`, filter at `:248`). So a
graph built node-by-node with guessed or copied positions cannot be re-flowed at all, even by a
compile round-trip. `material.authoring.auto_layout` is the only standalone layout verb
(`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:4451`).

**Fix:** add `anim.auto_layout` (the `anim.*` namespace holds `compile_agir` / `decompile_agir`)
following the shared contract in
[`F-blueprint-auto-layout-rpc`](F-blueprint-auto-layout-rpc.md) § API contract: required
`scope` (`all` | `nodes` | `selection` | `unpositioned`), optional graph name (default: every
graph `CollectAnimGraphs` returns, `PwGraphLayoutEdGraph.cpp:34`, including state-machine
graphs), `moved[]` read back after the write, `sizeSource`, and one `FScopedTransaction`
cancelled when nothing moved. Build the movable set per graph from `scope` and call
`PwGraphLayout::ArrangeEdGraph` (`Source/PinWright/Private/Layout/PwGraphLayoutEdGraph.h:48`);
`scope:"unpositioned"` over the whole blueprint is exactly `ArrangeAnimBlueprint`
(`PwGraphLayoutEdGraph.h:56`). Mark the package dirty; no compile, no save.

**Acceptance:** on an anim blueprint whose nodes sit at overlapping non-zero positions,
`anim.auto_layout {scope:"all"}` returns `movedCount > 0` and no overlapping node rects on
read-back; a second identical call returns `movedCount: 0` and opens no transaction; one undo
restores every `from` position; a missing `scope` is rejected by the required-parameter gate.

## History
- `#1-initial-spec` `OPEN` reporter — Anim auto-layout is only an AGIR-compile side effect; add a standalone anim.graph.auto_layout RPC that re-flows via FAGIRLayoutEngine (including state-machine inner graphs) without a compile round-trip, mirroring the shipped material.authoring.auto_layout (MaterialAuthoringHandler.cpp:3360), so agents can re-flow + re-measure after imperative node creation.
- `#2-re-rated` `OPEN` triage — Severity Medium -> Low. Graph layout is cosmetic (Low impact class) and a re-flow is already reachable through an AGIR compile; anim-graph re-layout is also a rare path, so nothing lifts it above Low.
- `#3-rephrased` `OPEN` developer — Old text targeted FAGIRLayoutEngine (deleted in 0a3b3acc when the PwGraphLayout core replaced it), cited material.authoring.auto_layout at a stale line, named the WONTFIX layout_report RPC as the re-measure step, kept a "Severity justification: Medium" section that #2 had overruled, and said a compile round-trip re-flows the graph. At 7230b41d the compile path calls PwGraphLayout::ArrangeAnimBlueprint, which only moves nodes at (0,0) (PwGraphLayoutEdGraph.cpp:248), so a compile does not re-flow positioned nodes. Retargeted to ArrangeEdGraph / ArrangeAnimBlueprint and the shared contract in F-blueprint-auto-layout-rpc; verb name follows the existing anim.* namespace. Severity stays Low. rice 1 1 1 2 -> 1 1 0.8 2: a feature from a gap analysis, not a session need, caps C at 0.8.
