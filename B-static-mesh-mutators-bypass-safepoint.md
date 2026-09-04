---
id: B-static-mesh-mutators-bypass-safepoint
title: "asset.nanite_rebuild_mesh and static_mesh.bake_transform run full StaticMesh rebuilds without a safe-point gate"
status: IN-REVIEW
severity: High
category: bug
tags: [static-mesh, nanite, bake-transform, safe-point, tick, compile, crash]
---

# Two StaticMesh rebuild verbs can execute inside an engine tick

## What's wrong

`asset.nanite_rebuild_mesh` changes Nanite settings, invokes the rebuild path, and drains static
mesh compilation at `AssetWorkflowHandler.cpp:1364-1392`. `static_mesh.bake_transform` calls
`PostEditChange` and `FinishCompilation` at `StaticMeshBakeTransformHandler.cpp:252-256` after
rewriting mesh descriptions, collision, and sockets. Neither verb is in
`Dispatch/SafePoint.cpp` and neither calls `RunAtSafePoint`.

These paths release/recreate render resources, enqueue and wait for compilation, and can flush
render work. If an RPC is drained during `UWorld::Tick`, the entire rebuild lands on that tick
stack. This is the same High crash mechanism already documented for StaticMesh rebuild verbs;
there is no direct kill reproduced through these two names, so severity is discounted from the
fatal impact class.

## What it should do

Add dispatcher safe-point entries for both verbs (neither has a cross-dispatch caller), and add a
source ratchet requiring every in-place StaticMesh build/`PostEditChange`/compilation drain to be
safe-pointed.

## Workaround

There is no reliable caller-side way to know whether the dispatcher is currently inside a tick.

## Related

- `B-nested-gamethread-marshal-defeats-tick-gate` fixed `asset.generate_lods` but not these verbs.
- `B-set-lod-settings-unguarded-rebuild` is the Geometry-module sibling.

## Fix

Root cause: Nanite rebuild and transform bake could execute their synchronous render-data
replacement directly from a tick-time handler stack, with no dispatcher or in-handler safe point.

Changed files:

- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRebuildRenderGuard.h` — shared
  `RunAtSafePoint` wrapper and live-consumer quiesce scope.
- `Plugins/PinWright/Source/PinWright/Private/Utils/MeshRenderConsumerScan.h` — shared target
  matcher for live `UStaticMeshComponent` and Niagara mesh-renderer consumers.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp` — routed
  `asset.nanite_rebuild_mesh` through the helper.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/StaticMeshBakeTransformHandler.cpp`
  — routed `static_mesh.bake_transform` through the helper.
- `Plugins/PinWright/Source/PinWright/Private/Dispatch/SafePoint.cpp` — added both verbs to the
  tick-unsafe registry.
- `Plugins/PinWright/Source/PinWrightGeometry/PinWrightGeometry.Build.cs` — explicit Niagara module
  dependency for the shared consumer matcher.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestStaticMeshDescribeHandler.cpp` —
  target-aware consumer readback coverage.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Infra/TestHandlerTickSafetyRatchet.cpp` —
  structural shared-guard route coverage.
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestSafePointGate.cpp` — structural
  tick-unsafe registry coverage.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Tests/Model/TestModelRebuildRenderGuard.cpp`
  — live component quiesce, shared Build/PostEditChange, and Niagara matcher coverage.
- `Plugins/PinWright/Docs/wiki-src/static_mesh.md`, `asset.md`, and `model.md` — guarded rebuild
  route and refusal contracts.

Tests: `PinWright.core.safe_point.KnownVictimsAreGated` and
`PinWright.infra.tick_safety.HandlerHazardsStayGated` structurally pin the registry and shared
route (not run per task restrictions); live render-state coverage is
`PinWright.Model.RebuildRenderGuard.QuiescedConsumersLoseAndRegainRenderState`, the shared route
is covered by `PinWright.static_mesh.rebuild_guard.SharedHelperResolvesMeshAtSafePoint`, and exact
Niagara slot matching by `PinWright.Model.RebuildRenderGuard.NiagaraMeshRendererMatchesTargetMesh`;
the Niagara diagnostic table and readback are pinned by
`PinWright.Model.RebuildRenderGuard.NiagaraMeshRendererIsAScannedCandidateClass` and
`PinWright.static_mesh.describe.ReportsLiveRenderConsumers`.

Deliberately unchanged: the separate `render.nanite_rebuild_mesh` continuation and unrelated
StaticMesh creation routes, which have their own tickets and response contracts. Enabled Niagara
systems are matched through emitter mesh-renderer properties; unresolved runtime mesh bindings
remain conservative candidates.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the tick-unsafe catalog; no editor, build, test, or RPC run was performed.
- `#2-shared-static-mesh-safepoint` `IN-REVIEW` developer — Added shared safe-point routing for Nanite rebuild and transform bake; source-only verification, no build or runtime test.
- `#3-target-mesh-static-consumers` `IN-REVIEW` developer — Extended the shared safe-point guard to explicitly quiesce target-mesh `UStaticMeshComponent` instances as well as Niagara consumers; source-only verification, no build or runtime test.
- `#4-target-niagara-mesh-matching` `IN-REVIEW` developer — Replaced the class-wide Niagara scan with enabled-system emitter mesh-renderer property matching, conservatively retaining unresolved runtime bindings; added the direct matcher test and source-only verification.
