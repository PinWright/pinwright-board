---
id: B-volume-brush-test-destroys-host-volume-on-label-collision
title: "The blocking-volume brush test can inspect and destroy a host volume selected by a fixed non-unique label instead of its spawned fixture"
status: OPEN
severity: High
category: bug
tags: [tests, volume, host-safety, actor-label, wrong-target, teardown, false-green]
encounters: 1
lastSeen: 2026-09-03T23:13:40+03:00
---

# The blocking-volume brush test tears down by fixed label

## What happens

`volume.create_blocking_volume.BrushGeometryInitialized` uses the fixed label
`McpRegressionBlockingVolume`, invokes the production handler without capturing
its response, finds the first `ABlockingVolume` with that display label, inspects
it, and calls `Destroy()` on that pointer
(`Source/PinWright/Private/Tests/World/TestVolumeHandlers.cpp:565-648`).

Unreal permits duplicate actor labels
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/ActorEditor.cpp:1280-1282`) and
`SetActorLabel` does not reject a collision (`:1291-1315`). The production handler
spawns a new volume, applies the requested label, and returns
`AddActorVerification` with its canonical actor path
(`Source/PinWright/Private/Handlers/Volume/VolumeHandler.cpp:575-608`). If a host
blocking volume already has the fixed label, the test can inspect and destroy that
pre-existing actor while leaving the actual fixture in the map. A host volume with
the requested 300/400/500 extents and valid brush data can satisfy every assertion,
so even a wrong target is not guaranteed to turn the test red.

## Why it matters

A test run can delete a real blocking volume, leak its generated replacement, and
leave the user's open level dirty. The test response does not disclose which actor
was inspected or destroyed because it discards the handler's canonical identity.

Severity is **High**: this is destructive wrong-target mutation in shared host
state, reduced one band from Critical because it requires a collision with a
test-specific label.

## What should happen

Give the fixture a GUID-suffixed label, invoke with response capture, and resolve
the returned `actorPath` before every geometry assertion. Declare
`FScopedEditorWorldActorGuard` before the handler call and remove the raw
`Spawned->Destroy()` so the guard owns deselection, new-actor teardown, and dirty
flag restoration on every exit (`Tests/TestWorldUtils.h:58-137`). Add this handler
call to the guarded-spawn structural ratchet.

## Workaround

Run the test only in a disposable map with no `McpRegressionBlockingVolume` actor,
then discard any dirty map state.

## Related

- Catalog `dataloss/3 wrong-target-identity-or-fallback` and `falsesuccess/7 wrong-target-scope-or-identity` — a destructive first label match replaces canonical actor identity.
- Catalog `dataloss/8 cross-caller-global-state-leak`, `crash/13 test-fixture-lifetime-is-not-scoped`, and `falsesuccess/12 non-discriminating-tests` — the live-world fixture is unscoped and a compatible host volume can satisfy the assertions.
- `B-tests-spawn-live-world-no-guard` — owns the generic unguarded-spawn risk; this ticket owns the volume family's distinct wrong-target deletion.
- `B-geometry-duplicate-mesh-teardown-guard` — sibling use of the same scoped teardown fix.

## History

- `#1-fixed-label-volume-teardown` `OPEN` reporter — Source-only pattern scan of the requested test trees. Traced the test through `volume.create_blocking_volume`, `SpawnVolumeActor`, `AddActorVerification`, first-label lookup, geometry assertions, and destroy; verified the engine's non-unique-label contract. Deduped against the full board: no ticket names this test/fixed label, and the generic live-world-spawn ticket does not cover wrong-target destruction after discarding an available actor path. No build, test, editor, Saved-file access, plugin edit, existing-ticket edit, or commit was performed.
