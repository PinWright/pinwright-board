---
id: B-geometry-duplicate-mesh-teardown-guard
title: "The Geometry duplicate-mesh E2E destroys probe actors without deselecting them or restoring the host level's dirty state"
status: DONE
severity: Medium
category: bug
tags: [geometry, tests, teardown, editor-world, selection, dirty-state]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# The Geometry duplicate-mesh E2E lacks guarded world teardown

## What happens

`DestroyDuplicateMeshProbeActors()` scans the live editor world by label prefix and
calls `World->DestroyActor()` directly
(`Source/PinWrightGeometry/Private/Tests/Geometry/TestActorDuplicateMeshIntegrity.cpp:50-74`).
The test installs that bespoke cleanup through `ON_SCOPE_EXIT` at
`TestActorDuplicateMeshIntegrity.cpp:122-124`. It does not snapshot or restore the
persistent level's dirty flag and does not remove probe actors/components from the
editor selection before destruction.

## Why it matters

A suite run can leave the shared host map dirty, leak a stale selection handle, or
leave probe actors behind if teardown cannot find them. Severity is Medium: host
contamination is observed elsewhere for this fixture shape, while the selection-set
crash path is reachable but not directly reproduced here.

## What should happen

Install a scoped editor-world guard before any handler can spawn a probe. On every
exit it must deselect created actors/components, destroy every created actor, and
restore the level package's prior dirty state. Add a structural check covering the
Geometry module so this E2E cannot regress to bespoke raw destruction.

## Workaround

Run the test only in a disposable editor/map and discard any dirty map state after
the run.

## Related

- `B-suite-host-gc-crash-in-combined-group-run` — wave-6 parent report.
- `B-tests-spawn-live-world-no-guard` — covers the same guard class in the main
  module but explicitly excludes this Geometry E2E.
- `B-tests-destroy-host-assets` — records this specific fixture's accumulated actors
  and dirty startup map as a separate symptom.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed raw actor destruction at `TestActorDuplicateMeshIntegrity.cpp:50-74` and the bespoke scope exit at `:122-124`, with no deselection or level-dirty restoration. The existing `B-tests-spawn-live-world-no-guard` explicitly leaves the Geometry E2E outside its scope, so this is a scoped child rather than a duplicate. No suite, build, test, editor, or MCP call was run. Severity Medium because shared-host contamination is the established impact; no selection-set crash was reproduced through this test.
- `#2-guarded-geometry-teardown` `IN-REVIEW` developer — Still reproducible at review time: `TestActorDuplicateMeshIntegrity.cpp` still carried `DestroyDuplicateMeshProbeActors` (raw `World->DestroyActor`, no deselect, no dirty restore) behind `ON_SCOPE_EXIT`. Sibling grep over `PinWrightGeometry/Private/Tests` found the same bespoke raw destroy in `TestGeometryDeformerEchoesMeshCounts.cpp` (`DestroyDeformerProbeActors`) and in the shared `GeometryTestHelpers::DestroyActorsWithLabel` (`PinWright/Private/Tests/Geometry/GeometryTestHelpers.h`) that 31 geometry test files call. Fix: extracted the guard's deselect-actor-and-components-then-`EditorDestroyActor(..., false)` step into `DeselectAndDestroyEditorActor(UWorld*, AActor*)` in `Tests/TestWorldUtils.h`; `FScopedEditorWorldActorGuard`'s destructor and `DestroyActorsWithLabel` both route through it, so every geometry teardown deselects first. The duplicate-mesh E2E and the deformer echo test drop their bespoke helpers and declare `FScopedEditorWorldActorGuard` before the first dispatch (it deselects, destroys the probe plus every `<Label>N` duplicate, restores the level's dirty flag; constructed first so it runs after the level-lock/cvar restores). Structural check: `PinWright.infra.contract.EditorWorldSpawn.GeometryTeardownRoutesThroughGuard` (`Tests/Infra/TestFixtureOwnershipContracts.cpp`) scans every `PinWrightGeometry/Private/Tests` source plus `GeometryTestHelpers.h` (comments/strings neutralized) and fails on any raw `DestroyActor(` / `EditorDestroyActor(` / `->Destroy(`, requires `DestroyActorsWithLabel` to call `DeselectAndDestroyEditorActor`, and requires every `Dispatch(` in the two E2E `RunTest` bodies to sit under an active guard. Failure direction: restoring either bespoke helper or the old helper body re-introduces a raw destroy and fails the scan; moving the guard after the first dispatch fails the guard-before-dispatch check. Remaining gap filed separately: the 31 `DestroyActorsWithLabel` callers still do not restore the level dirty flag (`B-geometry-label-helper-callers-no-dirty-restore`). Compile-checked: UBT -SingleFile on TestFixtureOwnershipContracts.cpp, clang -fsyntax-only fastcheck on the two converted tests, TestMeshIOHandler.cpp (helper caller), TestGeometryTargetResolution.cpp and TestActorErgonomicsHandlers.cpp (guard users); no editor, build or automation run (manager owns the slot).
- `#3-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full, non-skipped: `PinWright.infra.contract.EditorWorldSpawn.GeometryTeardownRoutesThroughGuard` covers the structural check the ticket asked for. Every PinWrightGeometry test source plus GeometryTestHelpers.h is free of raw DestroyActor/EditorDestroyActor/->Destroy, `DestroyActorsWithLabel` routes through `DeselectAndDestroyEditorActor`, and every Dispatch in the two E2E RunTest bodies sits under an active `FScopedEditorWorldActorGuard`, which deselects, destroys the created actors and restores the level dirty flag. The converted E2Es also passed in the same run: `PinWright.actor.duplicate.NeverSilentlySubstitutesDynamicMesh`, `.MeshProbeReflectionChainIntact`, `.MeshProbeIgnoresNonDynamicMeshActor` and `PinWright.geometry.deformers.EchoMeshCounts`, and the full suite finished with no crash. Remaining gap, filed separately: the 31 `DestroyActorsWithLabel` callers do not restore the dirty flag (B-geometry-label-helper-callers-no-dirty-restore).
