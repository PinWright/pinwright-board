---
id: B-set-lod-settings-unguarded-rebuild
title: "geometry.set_lod_settings rebuilds a live UStaticMesh without the safe-point or render-consumer guards model.compile uses for the same fatal render-state hazard"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, set-lod-settings, static-mesh, rebuild, render-thread, niagara, safe-point, crash]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# `geometry.set_lod_settings` rebuilds render data on an ordinary handler stack

> **SOURCE-ONLY.** The crash path is established by the in-tree guard and engine call sequence; it
> was not executed during this scan.

`LODCollisionHandler.cpp:188-213` loads an existing `UStaticMesh`, mutates one source model, then
calls `StaticMesh->Build()` and `PostEditChange()` in place. The handler does not scan for live
components that cache the mesh's render data and is not listed in
`Source/PinWright/Private/Dispatch/SafePoint.cpp` (that table contains `model.compile`, but no
`geometry.*` entry).

The tree already documents why both gates are required. `MeshRebuildRenderGuard.h` says an in-place
`UStaticMesh` rebuild can leave non-standard scene proxies, including Niagara mesh renderers,
holding freed render data; the next gather can hit a render-thread `check()`. It also flushes render
commands, which is unsafe when the RPC was reached from a named-thread pump. `ModelCompileHandler.cpp:
1252-1284` scans live consumers, wraps the entire compile/save in `FQuiesceScope`, and refuses if a
consumer cannot be quiesced. `set_lod_settings` performs the same class of rebuild without either
protection.

## What should happen

Add `geometry.set_lod_settings` to the shared safe-point table and reuse
`PinWrightMeshRebuild::ScanForStaleRenderStateConsumers` plus `FQuiesceScope` around the mutation,
build, post-edit notification, and save. Refuse before mutation when any consumer remains live, and
name those consumers in the error. A test should prove the method is in the safe-point registry and
that an unquiescable live consumer prevents the build.

**Workaround:** close or deactivate every component using the target mesh and issue the call only
while the editor is idle. This reduces the risk but does not replace the missing safe-point gate.

## Related

`B-geometry-generate-lods-no-disk-write` owns persistence for this verb, not rebuild safety.
`B-model-compile-live-niagara-mesh-renderer-raytracing-assert` supplies the already-landed guard.

## Fix

Root cause: `geometry.set_lod_settings` mutated and rebuilt an occupied StaticMesh directly,
bypassing both the dispatcher safe point and the live render-consumer quiesce used by `model.compile`.

Changed files:

- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRebuildRenderGuard.h` — shared safe-point,
  consumer scan, refusal, render flush, and render-state recreation implementation.
- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRenderConsumerScan.h` — shared target
  matcher for live `UStaticMeshComponent` and Niagara mesh-renderer consumers.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/LODCollisionHandler.cpp`
  — routed the mutation, `Build`, `PostEditChange`, and save through the shared helper.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Model/ModelCompileHandler.cpp` —
  shared helper adoption for the existing model path.
- `Plugins/PinWright/Source/PinWrightGeometry/PinWrightGeometry.Build.cs` — explicit Niagara module
  dependency for the shared consumer matcher.
- `Plugins/PinWright/Source/PinWright/Private/Dispatch/SafePoint.cpp` — added the verb to the
  tick-unsafe registry.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestStaticMeshDescribeHandler.cpp` —
  target-aware consumer readback coverage.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Infra/TestHandlerTickSafetyRatchet.cpp` —
  structural shared-guard route coverage.
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestSafePointGate.cpp` — structural
  tick-unsafe registry coverage.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Tests/Model/TestModelRebuildRenderGuard.cpp`
  — live component quiesce, shared Build/PostEditChange, and Niagara matcher coverage.
- `Plugins/PinWright/Docs/wiki-src/static_mesh.md` and `model.md` — shared target-aware consumer
  and refusal contracts used by the Geometry rebuild route.

Tests: `PinWright.core.safe_point.KnownVictimsAreGated`,
`PinWright.infra.tick_safety.HandlerHazardsStayGated`, and
`PinWright.static_mesh.rebuild_guard.SharedHelperResolvesMeshAtSafePoint` (not run per task
restrictions); `PinWright.Model.RebuildRenderGuard.QuiescedConsumersLoseAndRegainRenderState`
covers the live component teardown/recreation window, and
`PinWright.Model.RebuildRenderGuard.NiagaraMeshRendererMatchesTargetMesh` covers exact Niagara
mesh-slot matching; `PinWright.Model.RebuildRenderGuard.NiagaraMeshRendererIsAScannedCandidateClass`
and `PinWright.static_mesh.describe.ReportsLiveRenderConsumers` pin the diagnostic/readback paths.

Deliberately unchanged: `B-geometry-generate-lods-no-disk-write` persistence ownership and
unrelated geometry creation/rebuild routes. The shared helper target-matches live
`UStaticMeshComponent` instances and inspects enabled Niagara system emitter mesh-renderer
properties and explicit mesh entries before this route mutates the mesh; unresolved runtime mesh
bindings remain conservative candidates.

## History
- `#1-source-pattern-scan` `OPEN` reporter — `geometry.set_lod_settings` calls `UStaticMesh::Build`/`PostEditChange` directly, is absent from the shared safe-point registry, and bypasses the live render-consumer quiesce used by `model.compile`. Source-only; no editor or build was run.
- `#2-shared-static-mesh-guard` `IN-REVIEW` developer — Routed `geometry.set_lod_settings` through the shared safe-point and live-consumer guard; source-only verification, no build or runtime test.
- `#3-target-mesh-static-consumers` `IN-REVIEW` developer — Extended the shared guard to explicitly enumerate and quiesce target-mesh `UStaticMeshComponent` instances alongside Niagara consumers; source-only verification, no build or runtime test.
- `#4-target-niagara-mesh-matching` `IN-REVIEW` developer — Replaced the class-wide Niagara scan with enabled-system emitter mesh-renderer property matching, conservatively retaining unresolved runtime bindings; added the direct matcher test and source-only verification.
