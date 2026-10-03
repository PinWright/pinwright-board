---
id: F-controlrig-auto-layout-rpc
title: "Standalone controlrig.auto_layout RPC: re-flow positioned rig-graph nodes with the PwGraphLayout RigVM adapter (CRIR compile only lays out nodes it creates without a position)"
status: OPEN
severity: Low
category: feature
tags: [layout, controlrig, crir, rpc, authoring]
blockedBy: [F-graph-layout-metrics-core]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
rice: [1, 1, 0.8, 2]
priority: 3
---

# Standalone controlrig.auto_layout RPC (re-flow an existing rig graph)

There is no RPC that re-flows a Control Rig graph. The only layout run is inside
`controlrig.compile_crir` (`runLayout`, default true;
`Source/PinWright/Private/Handlers/ControlRig/CRIRCompileHandler.cpp:59`), which passes
`PwGraphLayout::ArrangeRigVMGraph` only the nodes that compile created without an explicit
position (`Source/PinWright/Private/CRIR/CRIRCompiler.cpp:1169-1174`, `:1974-1989`); every
existing node is a fixed obstacle. So a rig graph built node-by-node with guessed or copied
positions cannot be re-flowed at all, even by a compile round-trip.
`material.authoring.auto_layout` is the only standalone layout verb
(`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:4451`).

**Fix:** add `controlrig.auto_layout` (next to `compile_crir` / `decompile_crir`) following
the shared contract in [`F-blueprint-auto-layout-rpc`](F-blueprint-auto-layout-rpc.md) § API
contract: required `scope` (`all` | `nodes` | `selection` | `unpositioned`), target graph
(default: the rig's main graph), `moved[]` read back after the write, `sizeSource`, and one
undoable step cancelled when nothing moved. Build the movable set from `scope` and call
`PwGraphLayout::ArrangeRigVMGraph(Graph, Movable, Controller)`
(`Source/PinWright/Private/Layout/PwGraphLayoutRigVM.h:33`). That function writes through
`Controller->SetNodePosition` without an undo bracket (`PwGraphLayoutRigVM.h:31-32`), so the
handler must open and close its own controller undo bracket. Mark the package dirty; no
compile, no save.

**Acceptance:** on a rig graph whose nodes sit at overlapping positions,
`controlrig.auto_layout {scope:"all"}` returns `movedCount > 0` and no overlapping node rects on
read-back; a second identical call returns `movedCount: 0` and records no undo step; one undo
restores every `from` position; a missing `scope` is rejected by the required-parameter gate.

## History
- `#1-initial-spec` `OPEN` reporter — ControlRig auto-layout is only a CRIR-compile side effect; add a standalone controlrig.graph.auto_layout RPC that re-flows via FCRIRLayoutEngine (keeping pre-positioned-obstacle awareness) without a compile round-trip, mirroring the shipped material.authoring.auto_layout (MaterialAuthoringHandler.cpp:3360); material already has its own, so none is filed for material.
- `#2-re-rated` `OPEN` triage — Severity Medium -> Low. Graph layout is cosmetic (Low impact class) and re-flow is reachable through a CRIR compile; ControlRig graph re-layout is a rare path, so nothing lifts it above Low.
- `#3-rephrased` `OPEN` developer — Old text targeted FCRIRLayoutEngine::RunLayout (deleted in 0a3b3acc when the PwGraphLayout core replaced it), cited material.authoring.auto_layout at a stale line, named the WONTFIX layout_report RPC as the re-measure step, kept a "Severity justification: Medium" section that #2 had overruled, and said a compile round-trip re-flows the graph. At 7230b41d compile_crir lays out only nodes it created without a position (CRIRCompiler.cpp:1171-1173), so a compile does not re-flow existing nodes. Retargeted to PwGraphLayout::ArrangeRigVMGraph (no undo bracket of its own) and the shared contract in F-blueprint-auto-layout-rpc. Severity stays Low. rice 1 1 1 2 -> 1 1 0.8 2: a feature from a gap analysis, not a session need, caps C at 0.8.
