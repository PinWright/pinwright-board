---
id: B-tests-leak-host-content
title: "Automation tests leave dirty /Game packages that an editor-wide save-all leaks into host Content"
status: IN-REVIEW
severity: Medium
category: bug
tags: [tests, hygiene, asset-cleanup]
encounters: 2
lastSeen: 2026-09-10T17:23:34Z
---

# Automation tests leave dirty /Game packages that an editor-wide save-all leaks into host Content

A full automation run leaves permanent `.uasset` litter in the host project's
`Content/` folder, showing up as untracked files in the host's git repo. Observed
after runs on the PDS host (2026-07-05 and 2026-07-10 timestamps):
`SequenceAddCameraPath_<GUID>.uasset`, `SequenceAddKeyframeLoc_<GUID>.uasset`,
`SeqListSectionsIncludeKeys_<GUID>.uasset` (at the Content **root**), plus
`PW_ProcFoliage_<GUID>_Spawner*.uasset` pairs — and a spurious dirty resave of the
host startup map `L_Core.umap`.

## Corrected mechanism (the original ticket mis-stated this)

The handlers do **not** save `.uasset` files to disk. `McpSafeAssetSave`
(`Utils/AssetUtils.cpp:220`) is **mark-dirty-only** — it explicitly does *not* save,
just `MarkPackageDirty()` + `AssetRegistryModule::AssetCreated()`. The real leak is:

1. Tests create **non-transient `/Game` packages** and leave them **dirty**:
   - **Sequencer** — `FScopedRegisteredSequence` (`Tests/Media/TestSequencerHandlers.cpp:53`)
     builds a `/Game/<Prefix>_<GUID>` package at the Content root; the mutating handlers
     it drives (`sequencer.add_camera`, `sequence.add_keyframe`, ...) dirty it. Its
     destructor only did `AssetDeleted` + `RemoveFromRoot` — it never cleared the dirty flag.
   - **Foliage** — `foliage.create_procedural` (`Handlers/Environment/FoliageHandler.cpp:929`)
     hardcodes `/Game/ProceduralFoliage` and marks the Spawner + FoliageType packages dirty;
     `FFoliageCreateProceduralReportsInstancesSpawnedTest` (`Tests/World/TestEnvironmentHandlers.cpp:1976`)
     guarded only the spawned volume actor and never cleaned up those packages.
2. The suite's own **editor-wide save-all** tests (`FEditorSaveAllRespondsSynchronouslyTest`,
   `FUiSaveAllNoCrashTest`) then flush **every** dirty package to disk — which is exactly what
   drops the GUID `.uasset` files and the dirty `L_Core.umap`.

So the trigger is dirty-package-plus-save-all, not "tests save to disk."

## Why the original `/Game/PinWrightTests` reroute prescription was wrong

The foliage packages are created at a **production-hardcoded** path
(`/Game/ProceduralFoliage`) inside the handler; a test cannot redirect where the handler
writes. Re-homing the sequencer fixture would only move where save-all dumps it. The correct
fix is to ensure the tests leave **no dirty package** for save-all to flush — the
`SetDirtyFlag(false)` + `CleanupTestAsset` pattern already used across the suite
(`TestUtils.h:461`, `TestUtils.h:631`).

## Merge vector dropped (already clean)

The `merge_actors` test (`Tests/EditorOps/TestDebugHandlers.cpp:760`) already calls
`CleanupTestAsset(MergedPackageName)` and its world guard restores the level dirty flag, so
its merged asset is torn down in-scope before any save-all. No change needed there.

Expected: sequencer/foliage tests leave no dirty `/Game` package behind, so a later
editor-wide save-all has nothing to flush into the host Content tree.

**Workaround:** manually delete the leaked files and revert dirty host maps after each run
(on the fix/test-workflow hosts the per-iteration `git reset --hard` + `git clean -fd`
already erases them; the durable bite is a manual dev checkout like the PDS host).
**Fix:** de-dirty the sequencer fixture package in `~FScopedRegisteredSequence`; add
scope-exit `CleanupTestAsset` + de-dirty of the `/Game/ProceduralFoliage/<name>_Spawner`
and `_FT_<i>` packages in the foliage test. No production-code change and no
`/Game/PinWrightTests` reroute required.

## History
- `#1-initial-repro` `OPEN` reporter — Found 30 leaked .uasset files + a dirty L_Core.umap in the PDS host repo after automation runs dated 2026-07-05/07-10; traced to sequencer, merge_actors, and procedural-foliage tests saving without cleanup (sequencer ones at Content root).
- `#2-reword` `IN-REVIEW` fuzz2 — Reworded (real mechanism: dirty non-transient /Game packages flushed by the suite's own editor-wide save-all tests, NOT handlers saving to disk — McpSafeAssetSave is mark-dirty-only). Shipped test-side fix: de-dirty the sequencer fixture package in ~FScopedRegisteredSequence (Tests/Media/TestSequencerHandlers.cpp) plus new regression test PinWright.sequencer.fixture.LeavesNoDirtyPackage; CleanupTestAsset + de-dirty the foliage handler's hardcoded /Game/ProceduralFoliage/<name>_Spawner and _FT_0 packages in PinWright.foliage.create_procedural.ReportsInstancesSpawned (Tests/World/TestEnvironmentHandlers.cpp). Merge_actors vector dropped (already CleanupTestAssets its merged asset, TestDebugHandlers.cpp:760); the /Game/PinWrightTests reroute rejected (inapplicable to handler-hardcoded paths). Severity kept Medium. Plugin compiles clean; both scoped tests pass green.
- `#3-l-core-resave-survives-fix` `IN-REVIEW` reporter — Additional evidence, status unchanged (verification is the tester's call). The `L_Core.umap` resave named in `#1` is NOT fixed by the `#2` per-fixture work (`1d41d670`, 2026-07-13): `Saved/Logs/Automation_PinWright_verify2.log` (2026-08-10, i.e. post-fix) shows `Saving Map: /Game/System/FrontEnd/Maps/L_Core` at `:21743` and `L_Core_BuiltData` at `:21752`, both inside the start/complete window of `PinWright.editor.save_all.RespondsSynchronously` (`:21740`/`:21799`) — the writer is that test invoking the real unfiltered `editor.save_all` (TestEditorHandlers.cpp:292/:310), not any one dirty fixture. The PDS host reproduced it again on 2026-09-10 (`M L_Core.umap`, `M L_Core_BuiltData.uasset`, `?? Content/PinWrightTests/` 368 KB). Filed `B-suite-run-dirties-host-repo` for the writer plus the missing suite-level `/Game/PinWrightTests` sweep, so this ticket can stay scoped to the sequencer/foliage dirt sources it fixed.
- `#4-narrowed-by-writer-guard` `IN-REVIEW` developer — NARROWED, not closed; status unchanged (verification is the tester's call). `B-suite-run-dirties-host-repo`'s fix wraps the `editor.save_all` invocation in `Tests/EditorOps/TestEditorHandlers.cpp` in `PinWrightSuiteMaintenance::FScopedForeignDirtyPackageSuspension`, which de-dirties every dirty package outside `/Game/PinWrightTests` for the duration of the call and restores the flags after, so the editor-wide save-all named as the flushing mechanism in `#2` can no longer write ANY package outside the scratch root — including the `L_Core.umap` / `L_Core_BuiltData.uasset` resave that `#3` proved survived `1d41d670`. That closes the `L_Core` half of `#1` at the writer. What is NOT closed here: the leak SOURCES this ticket owns (a test leaving a dirty non-scratch `/Game` package behind) are unchanged — the guard stops the save, not the dirtying, and `FUiSaveAllNoCrashTest` named in `#2` has no live invocation in the tree today (the only `editor.save_all` invoke site is the one now guarded). If the tester agrees the remaining scope is only "tests must not leave dirty `/Game` packages", this ticket is about the sequencer/foliage fixtures alone and its severity could drop.
- `#5-correcting-4s-wording` `IN-REVIEW` developer — Correction to `#4`, status unchanged. `#4` said the leak SOURCES this ticket owns "are unchanged", which overclaims in the wrong direction: it reads as a statement about their current state when it is only a statement about what my work touched. Read it as **untouched by this work; `#2`'s sequencer/foliage fixes are shipped but still pending verification** — this session did not re-measure them, and nothing here should be taken as evidence for or against them. The rest of `#4` stands: the writer guard closes the `L_Core` flush vector, not the dirtying.
