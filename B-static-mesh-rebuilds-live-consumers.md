---
id: B-static-mesh-rebuilds-live-consumers
title: "Three main-module StaticMesh mutators rebuild render data without quiescing live Niagara mesh consumers"
status: IN-REVIEW
severity: High
category: bug
tags: [static-mesh, nanite, lod, bake-transform, niagara, render-state, raytracing, crash]
---

# StaticMesh rebuilds leave non-StaticMeshComponent proxies on freed render data

## What's wrong

The main module already documents the hazard in `Utils/MeshRenderConsumerScan.h`: Niagara scene
proxies cache a StaticMesh's render data and the engine does not reregister them during an in-place
mesh rebuild. `model.compile` uses `PinWrightMeshRebuild::FQuiesceScope` to destroy/flush/recreate
those consumers; `static_mesh.describe` reports the same candidate set.

Three scoped mutators do not use that guard:

- `asset.generate_lods`: `Mesh->Build()` + `PostEditChange()` at
  `AssetWorkflowHandler.cpp:1201-1202`.
- `asset.nanite_rebuild_mesh`: Nanite notification/build + compilation drain at `:1364-1392`.
- `static_mesh.bake_transform`: full `PostEditChange` rebuild + drain at
  `StaticMeshBakeTransformHandler.cpp:252-256`.

Concrete path: a live Niagara mesh renderer holds the target's old `FStaticMeshRenderData`; one of
these verbs rebuilds and frees it; the next ray-tracing gather dereferences the stale proxy. That
path has already produced a render-thread fatal through `model.compile`
(`B-model-compile-live-niagara-mesh-renderer-raytracing-assert`).

## What it should do

Move/share the quiesce half where the main module can use it, bracket the complete rebuild and
save, and refuse with the established not-quiescable error if any live consumer retains render
state. Do not merely add a tick safe point; position and consumer lifetime are separate guards.

## Workaround

Remove or deactivate every live Niagara component before using these mutators.

## Related

- `B-set-lod-settings-unguarded-rebuild` covers Geometry-module LOD setters, not these verbs.

## Fix

Root cause: the three main-module mutators rebuilt StaticMesh render data without sharing the
existing live-consumer teardown and safe-point sequencing used by `model.compile`.

Changed files:

- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRebuildRenderGuard.h` — shared safe-point,
  scan, refusal, flush, and render-state recreation helper.
- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRenderConsumerScan.h` — shared target
  matcher for live `UStaticMeshComponent` and Niagara mesh-renderer consumers.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp` — routed
  `asset.generate_lods` and `asset.nanite_rebuild_mesh` through the helper.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/StaticMeshBakeTransformHandler.cpp`
  — routed `static_mesh.bake_transform` through the helper.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/StaticMeshDescribeHandler.cpp` — made
  `static_mesh.describe` report the same target-aware consumer set.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Model/ModelCompileHandler.cpp` —
  shared `model.compile` adoption through the main-module helper.
- `Plugins/PinWright/Source/PinWrightGeometry/PinWrightGeometry.Build.cs` — explicit Niagara module
  dependency for the shared consumer matcher.
- `Plugins/PinWright/Source/PinWright/Private/Dispatch/SafePoint.cpp` — tick-unsafe entries.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestStaticMeshDescribeHandler.cpp` —
  target-aware consumer readback coverage.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Infra/TestHandlerTickSafetyRatchet.cpp` —
  structural shared-guard route coverage.
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestSafePointGate.cpp` — structural
  tick-unsafe registry coverage.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Tests/Model/TestModelRebuildRenderGuard.cpp`
  — live component quiesce, shared Build/PostEditChange, and Niagara matcher coverage.
- `Plugins/PinWright/Docs/wiki-src/static_mesh.md`, `asset.md`, and `model.md` — target-aware
  consumer, guarded-route, and refusal contracts.

Tests: `PinWright.static_mesh.rebuild_guard.SharedHelperResolvesMeshAtSafePoint`,
`PinWright.Model.RebuildRenderGuard.QuiescedConsumersLoseAndRegainRenderState`,
`PinWright.Model.RebuildRenderGuard.NiagaraMeshRendererMatchesTargetMesh`,
`PinWright.Model.RebuildRenderGuard.NiagaraMeshRendererIsAScannedCandidateClass`,
`PinWright.static_mesh.describe.ReportsLiveRenderConsumers`,
`PinWright.core.safe_point.KnownVictimsAreGated`, and
`PinWright.infra.tick_safety.HandlerHazardsStayGated` (not run per task restrictions).

Deliberately unchanged: unrelated `render.nanite_rebuild_mesh` and non-StaticMesh rebuild routes
remain outside these tickets. The helper explicitly target-matches live `UStaticMeshComponent`
instances and inspects enabled Niagara system emitter mesh-renderer properties and explicit mesh
entries; unresolved runtime mesh bindings remain conservative candidates, so neither consumer
family is left in the rebuild window.

## History
- `#1-pattern-scan` `OPEN` reporter — Source and existing-crash-path confirmation only; no editor, build, test, or RPC run was performed.
- `#2-shared-static-mesh-guard` `IN-REVIEW` developer — Routed all three main-module StaticMesh rebuilds through the shared safe-point and live-consumer guard; source-only verification, no build or runtime test.
- `#3-target-mesh-static-consumers` `IN-REVIEW` developer — Added explicit target-mesh enumeration and quiescing for live `UStaticMeshComponent` instances alongside the Niagara consumer scan; source-only verification, no build or runtime test.
- `#4-target-niagara-mesh-matching` `IN-REVIEW` developer — Replaced the class-wide Niagara scan with enabled-system emitter mesh-renderer property matching, conservatively retaining unresolved runtime bindings; added the direct matcher test and source-only verification.
