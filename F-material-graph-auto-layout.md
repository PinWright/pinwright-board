---
id: F-material-graph-auto-layout
title: "Standalone auto-layout RPC for material graphs"
status: DONE
severity: Low
category: feature
tags: [material, mgir, layout, authoring, ergonomic]
---

# Standalone auto-layout RPC for material graphs

Material-graph auto-layout is reachable only as a side effect of
`material.compile_mgir { runLayout: true }` (default `true`). After a
sequence of imperative `material.graph.add_expression` /
`material.graph.connect` calls — where x/y were guessed, copied from a
template, or omitted entirely — there is no way to ask the editor to
re-flow the graph without going through the full MGIR compile/decompile
round-trip.

The underlying engine is already a clean, target-agnostic helper:
`FMGIRLayoutEngine::Layout(UMaterial*)` and
`::Layout(UMaterialFunction*)` in
`Source/EditorAutomationRpcGateway/Private/MGIR/MGIRLayoutEngine.cpp`.
`MGIRCompiler.cpp` only calls these inside `FinalizeMaterial` /
`FinalizeMaterialFunction` when `Options.bRunLayout` is true. Exposing
that one helper as its own RPC is cheap and isolates a capability MGIR
currently monopolizes.

Parallel to `F-niagara-compile-save-explicit`, which split standalone
`niagara.compile` / `niagara.save` out of edit-RPC side-effect flags.

## Why it matters

Workflow shapes that need this:

1. **Imperative authoring cleanup.** Agent builds a material via
   per-node `material.graph.add_expression` with rough x/y, then wants
   the editor to re-flow before save — no compile round-trip needed.
2. **Hand-edited or imported graphs.** Author or external tool produces
   a material with overlapping or off-canvas expressions. Running
   `material.compile_mgir` just to lay out is heavy: it re-parses,
   re-resolves, re-compiles shaders.
3. **Material functions.** Same need for `UMaterialFunction` graphs;
   the layout engine already handles them.
4. **Composite graphs.** MGIR compile currently rejects composite
   subgraphs (`MGIR_COMPOSITE_NOT_SUPPORTED`,
   `B-mgir-composite-subgraph-compile-fail`), so re-running compile is
   not a workaround for those assets. A standalone layout RPC works
   regardless of compile path.

## Workaround today

- `material.decompile_mgir` + `material.compile_mgir mode:"replace"
  runLayout:true` — heavy: full round-trip with shader recompile and
  data loss on any compile-unsupported feature.
- No-op `material.compile_mgir mode:"append"` with an empty IR body —
  cleaner but still triggers `ForceRecompileForRendering`.
- Manual editor formatting via UI — not accessible from automation.

## Proposal

```
material.authoring.auto_layout(
    assetPath: string           // UMaterial or UMaterialFunction
) -> {
    target: string,             // resolved asset path
    expressionsLaidOut: number,
    durationMs: number
}
```

Implementation: dispatch on resolved asset type, call
`FMGIRLayoutEngine::Layout(Material)` or `::Layout(Function)`, then
`PreEditChange` / `PostEditChange` / `MarkPackageDirty` on the asset
(skip `ForceRecompileForRendering` — layout-only does not invalidate
shaders). No `save` flag; let the caller invoke `editor.save_all`
explicitly. Live next to existing `material.authoring.*` handlers in
`MaterialAuthoringHandler.cpp`.

Out of scope for this ticket:

- Layout-quality tuning (see `B-node-layout-poor` history — closed).
- Per-subgraph or selection-based layout. The engine helper is
  whole-graph; partial layout is a separate ergonomic ticket if ever
  asked for.
- AGIR / BPIR equivalents. `FAGIRLayoutEngine` exists with the same
  shape; once material lands, mirror as `anim.authoring.auto_layout`
  in a follow-up if there is demand.

## Cross-ref

- `F-niagara-compile-save-explicit` — same pattern: lift capability
  out of edit-RPC side-effect flag.
- `F-mgir-material-graph-ir` — original MGIR feature where `runLayout`
  was first introduced.
- `B-node-layout-poor` (DONE) — layout-quality bug, distinct from
  exposing the capability.
- `B-mgir-composite-subgraph-compile-fail` — explains why the
  compile-round-trip workaround does not cover composite graphs.

## History
- `#1-no-standalone-material-layout` `OPEN` reporter — Confirmed via grep: `FMGIRLayoutEngine::Layout` is called only from `MGIRCompiler.cpp` `FinalizeMaterial` / `FinalizeMaterialFunction` behind `Options.bRunLayout`. No `material.authoring.auto_layout` or similar RPC registered (no hits in `Source/Handlers/Material/` outside `MGIRCompileHandler.cpp`'s `runLayout` param). Layout helper is already target-agnostic and works on both `UMaterial` and `UMaterialFunction`. Cheap to expose; lets agents re-flow after imperative `material.graph.add_expression` chains without paying for MGIR decompile+recompile+shader rebuild.
- `#2-standalone-auto-layout-rpc` `IN-REVIEW` developer — Added material.authoring.auto_layout(assetPath) appended to MaterialAuthoringHandler.cpp. Dispatches on UMaterial vs UMaterialFunction, calls FMGIRLayoutEngine::Layout, does PreEditChange/PostEditChange/MarkPackageDirty without ForceRecompileForRendering (layout-only doesn't invalidate shaders). Returns {target, expressionsLaidOut, durationMs}. Regression test exercises the engine helper directly on a transient material with overlapping (0,0) constants and asserts positions change after the call.
- `#3-verify-fix` `DONE` tester — Verified: `material.authoring.auto_layout?` schema returns the expected single required `assetPath` param. Called on `/Game/UI/Hud/M_MSDF_Icon` (UMaterial) → `{target, expressionsLaidOut:6, durationMs:0.88}`; called on `/Game/UI/Hud/Art/MF_UI_RadialGrad0To1` (UMaterialFunction) → `{expressionsLaidOut:8, durationMs:0.02}` — both dispatch paths work, response shape matches spec. Missing-asset path returns `ASSET_NOT_FOUND` cleanly.
