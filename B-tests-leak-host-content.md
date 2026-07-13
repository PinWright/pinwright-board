---
id: B-tests-leak-host-content
title: "Automation tests leave dirty /Game packages that an editor-wide save-all leaks into host Content"
status: IN-REVIEW
severity: Medium
category: bug
tags: [tests, hygiene, asset-cleanup]
claimedBy: fuzz2
claimedAt: 2026-07-13T08:21:32.9389776+03:00
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
- `#2-reword` `IN-REVIEW` fuzz2 — Reworded: real mechanism is dirty non-transient /Game packages flushed by the suite's own editor-wide save-all tests, NOT handlers saving to disk (McpSafeAssetSave is mark-dirty-only). Scope narrowed to (a) de-dirty the sequencer fixture package and (b) CleanupTestAsset + de-dirty the foliage handler's hardcoded /Game/ProceduralFoliage packages at their existing paths; the /Game/PinWrightTests reroute is inapplicable to handler-hardcoded paths. Merge_actors vector dropped — that test already CleanupTestAssets its merged asset (TestDebugHandlers.cpp:760). Severity kept Medium. Test-side only.
