---
id: B-geometry-label-helper-callers-no-dirty-restore
title: "31 geometry test files tear down probes with GeometryTestHelpers::DestroyActorsWithLabel and never restore the level package's dirty flag"
status: OPEN
severity: Low
category: bug
tags: [geometry, tests, teardown, editor-world, dirty-state]
encounters: 1
lastSeen: 2026-10-02T12:00:00+03:00
rice: [1, 1, 1, 2]
priority: 4
---

# Geometry label-helper teardown leaves the editor level dirty

## What happens

`GeometryTestHelpers::DestroyActorsWithLabel`
(`Source/PinWright/Private/Tests/Geometry/GeometryTestHelpers.h`) is the teardown for
31 files under `Source/PinWrightGeometry/Private/Tests/Geometry/` (~150 call sites,
~105 `RunTest` bodies; `grep -rl DestroyActorsWithLabel Source/PinWrightGeometry`).
Since `B-geometry-duplicate-mesh-teardown-guard` `#2` it deselects before destroying
(via `DeselectAndDestroyEditorActor`), but it is a post-hoc, label-matched cleanup: it
has no snapshot, so it cannot restore the persistent level package's dirty flag, and
every exit path must remember to call it.

## Why it matters

Each run of these tests leaves the editor world's level package dirty. Since
`PinWright.aa_suite_start.OpenBlankTransientWorld` the full suite runs on an untitled
`/Temp` world, so host content is not reachable; a scoped run that skips
`aa_suite_start` runs on the host's startup map, where the dirty flag is then flushed
by any later `editor.save_all` or the editor's save prompt. Severity Low for that
reason.

## What should happen

Declare `FScopedEditorWorldActorGuard` at the top of each spawning `RunTest` (before
the first dispatch) and delete the matching `DestroyActorsWithLabel` calls, as done
for `TestActorDuplicateMeshIntegrity.cpp` and `TestGeometryDeformerEchoesMeshCounts.cpp`.
Watch for tests that load or swap maps (the guard holds a raw `UWorld*`). Then extend
`PinWright.infra.contract.EditorWorldSpawn.GeometryTeardownRoutesThroughGuard` to
require the guard in every geometry `RunTest` that dispatches a spawning verb, and
retire `DestroyActorsWithLabel`.

## Related

- `B-geometry-duplicate-mesh-teardown-guard` — fixed the two bespoke raw-destroy E2Es
  and routed the label helper through the deselecting destroy step.
- `B-tests-spawn-live-world-no-guard` — same conversion for the main module.

## History
- `#1-filed-label-helper-dirty-gap` `OPEN` developer — Found while fixing `B-geometry-duplicate-mesh-teardown-guard`: converting ~105 RunTests across 31 files in a tree shared with parallel agents was out of that ticket's scope, so the shared helper was fixed for the selection hazard only and the dirty-flag restore is filed here. Source-only; nothing was run.
