---
id: B-water-tests-destroy-host-lake-on-label-collision
title: "Water tests can destroy a host lake selected by a fixed non-unique label while leaking the fixture and still passing cleanup checks"
status: OPEN
severity: High
category: bug
tags: [tests, water, host-safety, actor-label, wrong-target, teardown, false-green]
encounters: 1
lastSeen: 2026-09-03T23:13:40+03:00
---

# Water tests tear down the first lake with a fixed label

## What happens

`water.spawn_water_body.SpawnsLakeAndDestroys` records the live-world lake count,
asks the production handler to spawn `McpTestLake`, discards the response, then
selects the first `AWaterBodyLake` whose display label equals that literal and
destroys it (`Source/PinWright/Private/Tests/World/TestWaterHandlers.cpp:74-115`).
It declares cleanup successful solely when the final count equals the original
count (`:117-124`).

Actor labels are explicitly non-unique in Unreal
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/ActorEditor.cpp:1280-1282`), and
`SetActorLabel` stores the requested label without collision rejection
(`:1291-1315`). If the host map already contains an `AWaterBodyLake` labelled
`McpTestLake`, the handler creates a second same-label lake. The test may select
and destroy the host lake: the count still returns from N+1 to N, so the cleanup
assertion passes while the new fixture remains and the original actor is gone.

`water.set_water_body_underwater_post_process.RejectsUnmatchedSettingsKeys` repeats
the same fixed-label first-match teardown with `McpUnderwaterDropLake`
(`TestWaterHandlers.cpp:145-175,207`). On collision it can also act on or destroy
the wrong lake, irrespective of whether the response assertions catch the target
ambiguity.

The production spawn already sends `AddActorVerification`, including a canonical
actor path (`Handlers/Water/WaterHandler.cpp:163-179`); neither test captures it.

## Why it matters

A test run can delete a real lake from the user's open map, leak a generated water
body, and leave the level dirty. The first test's count-only postcondition masks
the actor substitution, so this destructive outcome can be reported green and
later persisted by a map save.

Severity is **High**: the impact is host-world data loss and a false-green test,
reduced one band from Critical because it requires a collision with a test-like
label and first-match iteration to choose the pre-existing lake.

## What should happen

Use a GUID-suffixed requested label, `InvokeHandlerWithCapture`, and the returned
`actorPath` to resolve and inspect the exact spawned lake. Declare
`FScopedEditorWorldActorGuard` before the handler call and let it remove every
new actor, deselect it, and restore prior level dirtiness on every exit
(`Tests/TestWorldUtils.h:58-137`). The cleanup assertion must verify the original
actor set by identity, not only an unchanged class count. Add both handler calls
to the guarded-spawn structural ratchet.

## Workaround

Run the two tests only in a disposable map with no same-label lakes, then discard
the map's dirty state after the run.

## Related

- Catalog `dataloss/3 wrong-target-identity-or-fallback` and `falsesuccess/7 wrong-target-scope-or-identity` — teardown chooses the first non-unique label match instead of the response path.
- Catalog `dataloss/8 cross-caller-global-state-leak`, `crash/13 test-fixture-lifetime-is-not-scoped`, and `falsesuccess/12 non-discriminating-tests` — the shared-world fixture is unscoped and count equality cannot identify substitution.
- `B-tests-spawn-live-world-no-guard` — covers the general unguarded-spawn mechanism; this ticket owns the distinct wrong-target deletion and false-green count in the water family.
- `B-geometry-duplicate-mesh-teardown-guard` — sibling test teardown conversion to the same scoped guard.

## History

- `#1-label-collision-destroys-host-lake` `OPEN` reporter — Source-only pattern scan of the requested test trees. Traced both test entries through `water.spawn_water_body`, its active-world spawn, response verification, label lookup, destroy, and final assertion; verified from UE 5.8 engine source that labels may collide. Deduped against the full board: no ticket names either water test or the label-collision substitution; `B-tests-spawn-live-world-no-guard` owns generic fixture leakage, not deletion of a pre-existing same-label actor while the count test passes. No build, test, editor, Saved-file access, plugin edit, existing-ticket edit, or commit was performed.
