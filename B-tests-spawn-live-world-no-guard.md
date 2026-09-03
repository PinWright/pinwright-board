---
id: B-tests-spawn-live-world-no-guard
title: "12 test files spawn actors into the live editor world through SpawnActorInActiveWorld with no FScopedEditorWorldActorGuard, so probe actors accumulate in the host's map and the selection-set fatal the guard exists to prevent stays reachable"
status: IN-REVIEW
severity: Medium
category: bug
tags: [tests, hygiene, editor-world, actor-cleanup, scoped-guard, host-safety, selection-set, fatal]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# The guard exists, names the fatal it prevents, and 12 spawning test files do not use it

`FScopedEditorWorldActorGuard` (`Source/PinWright/Private/Tests/TestWorldUtils.h:58-135`)
snapshots the editor world's actor set and the persistent level's dirty flag on construction,
and on destruction destroys every actor spawned during the scope, **deselects each one first**,
and restores the dirty flag. Its own comment states why the deselect is mandatory and not
tidiness:

> Deselect before destroying. `EditorDestroyActor` does not remove the actor from the editor
> selection, so an actor that a test (or a handler like `editor.focus_actor`, which calls
> `GEditor->SelectActor`) left selected keeps a stale typed-element handle in `USelection`
> after it is gone. The next code path that walks the selection — e.g. the transform-widget
> helper run during `editor.save_all`'s content validation — then dereferences the dead element
> and hits the fatal **"Element type ID '0' has not been registered!"** assert.

That is a suite-killing fatal, and `editor.save_all` is exercised by the suite itself.

## Census

17 test files call `SpawnActorInActiveWorld` (the helper at `TestWorldUtils.h` that spawns into
`GEditor->GetEditorWorldContext().World()` — the host's real map, not a scratch world). Five of
them use the guard. **Twelve do not:**

```
Source/PinWright/Private/Tests/Environment/TestEnvironmentDirtyFlags.cpp
Source/PinWright/Private/Tests/Sequencer/TestSequencerAddActorBinding.cpp
Source/PinWright/Private/Tests/Sequencer/TestSequencerBakeControlRig.cpp
Source/PinWright/Private/Tests/Sequencer/TestSequencerCameraRigBinding.cpp
Source/PinWright/Private/Tests/Sequencer/TestSequencerControlRigTrack.cpp
Source/PinWright/Private/Tests/Sequencer/TestSequencerFbxRoundtrip.cpp
Source/PinWright/Private/Tests/Sequencer/TestSequencerRemoveActorsNameLabel.cpp
Source/PinWright/Private/Tests/Sequencer/TestSequencerSectionRangeUnits.cpp
Source/PinWright/Private/Tests/World/TestActorDuplicateComponentHandler.cpp
Source/PinWright/Private/Tests/World/TestActorDuplicateMeshIntegrity.cpp
Source/PinWright/Private/Tests/World/TestCreateProceduralTerrainLabel.cpp
Source/PinWright/Private/Tests/World/TestGetComponentsLargePayload.cpp
```

The five that do (`TestAnimationShotsSubjects`, `TestSequencerComponentBinding`,
`TestSequencerRepointActor`, `TestPropertyResolverAncestorFallback`,
`TestEnvironmentHandlers`) prove the idiom is established and applicable to exactly these
call shapes, including inside the sequencer group.

## The consequence is already measured

`B-tests-destroy-host-assets` (IN-REVIEW, Critical) recorded it as its "second symptom":
`TestActorDuplicateMeshIntegrity.cpp` left **20 distinct `PW_DupMeshIntegrity_<guid>` actor
labels accumulated in `Content/Maps/ExampleProjectWelcome.umap`** — one per suite run — on a
map that is that host's `EditorStartupMap`, `GameDefaultMap` *and* `ServerDefaultMap`. It rolls
a bespoke `ON_SCOPE_EXIT` cleanup that, unlike the guard, never restores the level's dirty flag,
so the map stays dirty for the suite's own `editor.save_all` tests to flush.

That ticket listed the conversion as fix item 4 and then **did not do it**, routing it to
`B-tests-leak-host-content`:

> Item 4 (`TestActorDuplicateMeshIntegrity`) NOT attempted: owned by
> `B-tests-leak-host-content`.

`B-tests-leak-host-content` (IN-REVIEW, Medium) does **not** own it: its shipped `#2` fix
covers only the sequencer fixture package and the procedural-foliage packages, and its subject
is dirty `/Game` *packages*, not leaked *actors*. So the item is currently owned by nobody, and
the two files named in this session's independent audit (`TestActorDuplicateMeshIntegrity.cpp`
and `TestActorDuplicateComponentHandler.cpp`) are 2 of the 12 above.

The path note in `B-tests-destroy-host-assets` is only partly stale: the current tree still has
the end-to-end fixture at `PinWrightGeometry/Private/Tests/Geometry/TestActorDuplicateMeshIntegrity.cpp`
and also has the pure probe/classifier coverage at
`PinWright/Private/Tests/World/TestActorDuplicateMeshIntegrity.cpp`. The 12-file list here names
the main-module pure test; the geometry E2E test remains outside this ticket's main-module scope.

Provenance for the wider census: `B-suite-host-gc-crash-in-combined-group-run` `#4` recorded
"11 test files spawn into the live editor world without `FScopedEditorWorldActorGuard`
(including a second actor-group file, `TestActorDuplicateComponentHandler.cpp`, which the
earlier note missed)" as an incidental finding while refuting that ticket's own hypothesis. A
finding in another ticket's history is never scheduled by the picker, so it is filed here. The
count is **12**, not 11, on a re-derivation over all eight modules.

## Severity

**Medium.** Two impact classes are in play and neither reaches higher on the evidence held:

- *Observed:* actors accumulate in the host's default startup map and the level package is left
  dirty for a save-all to flush. That is host litter — reversible by deleting actors or
  reverting the map — which is the same class and the same rating as
  `B-tests-leak-host-content` (Medium).
- *Reachable but unobserved:* the "Element type ID '0'" fatal. It would be Critical (editor
  crash), but **no crash has been attributed to it**; the guard's comment describes the
  mechanism, and no crash report in this checkout names it. Rating on an unobserved crash would
  be inflation.

Reach: every full suite run on every host, which argues a bump up from the litter class — but
Medium is already where the equivalent litter ticket sits with the same reach, so no further
bump is applied.

**Escalation condition:** a crash report or suite abort naming
`"Element type ID '0' has not been registered!"`, or a selection-walk fault during
`editor.save_all`, moves this to Critical immediately.

**Fix:** put `FScopedEditorWorldActorGuard` at the top of each `RunTest` body in the 12 files —
it is default-constructed and needs no arguments, so this is one line per test. For
`TestActorDuplicateMeshIntegrity.cpp`, either replace the bespoke
`DestroyDuplicateMeshProbeActors` scope-exit with the guard or make it restore the level dirty
flag (`LevelPackage->SetDirtyFlag(bLevelWasDirty)`); the guard is preferable because it also
brings the deselect. Consider a `TestWikiSrcOverlayStructure`-style infra lint that fails when
a test file calls `SpawnActorInActiveWorld` without declaring the guard — the whole class
recurs otherwise, as it already has twice.

## Fix

**Verdict: STATICALLY COMPLETE WITHIN THE TICKET SCOPE.** The root cause is confirmed: the 12 named main-module test files
either spawn into the active editor world directly or invoke a test handler that does so
indirectly, and their probe actors were not consistently protected by
`FScopedEditorWorldActorGuard`. The literal census overstates direct helper calls: comments in
`TestEnvironmentDirtyFlags.cpp` and `TestCreateProceduralTerrainLabel.cpp` mention the helper,
while the actual spawn is inside the invoked production handler. The current checkout also
retains both duplicate-mesh files; the geometry E2E path was not moved and was not edited.

The 12 named tests now declare `FScopedEditorWorldActorGuard` before their live-world fixture
work, and the duplicate-mesh pure test no longer uses its bespoke actor-destroy scope exits.
Three additional unguarded bodies found during review in
`Tests/World/TestEnvironmentHandlers.cpp` are also guarded: the two tests that call the local
`SpawnActorWithRootScene` helper and the directional-light test that calls
`SpawnActorInActiveWorld` directly. Their manual `Actor->Destroy()` scope exits were removed
because those ran before the world guard and prevented its deselection path from seeing the actor.
The indirect environment/terrain tests retain their existing dirty-flag assertions while the
guard owns actor teardown. The registered structural ratchet now parses each `RunTest` body after
neutralizing comments and literals. It requires an active lexical
`FScopedEditorWorldActorGuard` before each direct `SpawnActorInActiveWorld` call and each known
local actor-spawn helper call (including `SpawnActorWithRootScene`), and it names all four
handler-indirect `RunTest` bodies explicitly and requires the guard before their handler calls.
Missing, unreadable, or brace-malformed source fails instead of silently reducing coverage.

Changed source files:

- `Plugins/PinWright/Source/PinWright/Private/Tests/Environment/TestEnvironmentDirtyFlags.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerAddActorBinding.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerBakeControlRig.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerCameraRigBinding.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerControlRigTrack.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerFbxRoundtrip.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerRemoveActorsNameLabel.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerSectionRangeUnits.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestActorDuplicateComponentHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestActorDuplicateMeshIntegrity.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestCreateProceduralTerrainLabel.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestGetComponentsLargePayload.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Infra/TestFixtureOwnershipContracts.cpp`

Structural test ID: `PinWright.infra.contract.EditorWorldSpawn.GuardedTestSpawns`. It was not
run, and no build, Unreal, MCP, or runtime verification was performed as required by the brief.

Deliberate nonchanges: no production handlers, `TestWorldUtils`, `TestAssetTeardown`,
`Utils/AssetUtils`, or `PinWrightGeometry` files were edited. The geometry E2E bespoke cleanup
remains outside this ticket's named main-module set. This is static verification only; the
registered ratchet and guarded tests still require compile and automation execution by review.

## History
- `#1-twelve-unguarded-spawn-files` `OPEN` reporter — Source-only census over all eight modules; **no suite was run and no plugin source was modified** (tree is mid-verification on another wave). `grep -rl SpawnActorInActiveWorld` over `Source/` returns 17 test files and 9 handler files; `grep -rl FScopedEditorWorldActorGuard` returns 65 files; the set difference over the test files is the **12** listed above. Read the guard at `Tests/TestWorldUtils.h:58-135` and confirmed its destructor deselects from both `GEditor->GetSelectedActors()` and `GetSelectedComponents()` before `EditorDestroyActor`, and restores the persistent level's dirty flag last — and that its comment names the fatal ("Element type ID '0' has not been registered!") and the path that reaches it (`editor.save_all` content validation walking a stale typed-element handle). Widens `B-suite-host-gc-crash-in-combined-group-run` `#4`'s incidental finding of 11 files to a re-derived 12, and picks up the item `B-tests-destroy-host-assets` `#3` declined to do and mis-routed to `B-tests-leak-host-content`, whose shipped fix covers dirty packages only and not leaked actors — so the item was owned by nobody. Also corrects a stale path in `B-tests-destroy-host-assets`: `TestActorDuplicateMeshIntegrity.cpp` is now under `PinWright/Private/Tests/World/`, not `PinWrightGeometry/Private/Tests/Geometry/`. Filed separately from `B-tests-addtoroot-fixtures-never-unrooted` (the AddToRoot half of the same audit) because the mechanisms differ (world-actor teardown vs GC-root teardown), the fix sites differ, and merging would force one half to be worked at the other's priority. Severity Medium with the crash escalation condition recorded rather than pre-applied.
- `#2-guard-live-world-spawns` `IN-REVIEW` developer — Source-only PARTLY TRUE review; guarded the 12 named main-module test files and added `PinWright.infra.contract.EditorWorldSpawn.GuardedTestSpawns` with indirect-spawn baselines. No build or test was run. The remaining handler-focused direct spawn and geometry E2E cleanup are deliberate nonchanges recorded above.
- `#3-close-review-gaps` `IN-REVIEW` developer — Static review correction: guarded the two
  `SpawnActorWithRootScene` callers and the direct directional-light spawn in
  `Tests/World/TestEnvironmentHandlers.cpp`, removing their earlier manual destroy exits so the
  guard owns deselection and teardown. Replaced the whole-file count ratchet with registered
  per-`RunTest` lexical guard-before-spawn coverage in `TestFixtureOwnershipContracts.cpp`;
  unreadable/malformed files and missing explicit indirect bodies fail closed. Corrected the
  duplicate-mesh path evidence: both the main-module pure test and geometry E2E test exist. No
  build, Unreal, automation, MCP, or runtime verification was run.
