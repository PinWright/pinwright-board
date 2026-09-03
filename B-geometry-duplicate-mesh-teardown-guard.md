---
id: B-geometry-duplicate-mesh-teardown-guard
title: "The Geometry duplicate-mesh E2E destroys probe actors without deselecting them or restoring the host level's dirty state"
status: OPEN
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
