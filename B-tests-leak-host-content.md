---
id: B-tests-leak-host-content
title: "Automation tests leak saved assets into the host project's Content tree"
status: OPEN
severity: Medium
category: bug
tags: [tests, hygiene, asset-cleanup]
---

# Automation tests leak saved assets into the host project's Content tree

A full automation run leaves permanent `.uasset` litter in the host project's
`Content/` folder, which shows up as untracked files in the host's git repo and
risks being committed as junk. Observed after runs on the PDS host (2026-07-05
and 2026-07-10 timestamps):

- **Sequencer tests** (`Tests/Media/TestSequencerHandlers.cpp`) save GUID-named
  assets at the **Content root**: `SeqListSectionsIncludeKeys_<GUID>.uasset`,
  `SequenceAddCameraPath_<GUID>.uasset`, `SequenceAddKeyframeLoc_<GUID>.uasset`
  (15 files observed). No cleanup at all, and no dedicated subfolder.
- **merge_actors tests** (`Handlers/Debug/PerformanceHandler.cpp` default
  `/Game/Merged/MERGED_<FirstActor>` path) leave
  `Content/Merged/SM_MERGED_PW_MergeSrcA_<GUID>.uasset` behind (5 files).
- **Procedural foliage tests** (`Handlers/Environment/FoliageHandler.cpp:929`
  hardcodes `/Game/ProceduralFoliage`) leave
  `PW_ProcFoliage_<GUID>_Spawner*.uasset` pairs behind (10 files).

Related collateral: the host's startup map (`L_Core.umap`) came out of the same
sessions with a dirty resave (likely an editor-wide save-all during tests),
producing a spurious modification in the host repo.

Expected: tests that must save to disk use a dedicated test folder (e.g.
`/Game/PinWrightTests`, which `CleanupTestAsset` already serves elsewhere) and
delete their assets in `ON_SCOPE_EXIT`, including the on-disk package files.
Tests should never save at the `Content/` root, and should never trigger
save-all over host content.

**Workaround:** manually delete the leaked files and revert dirty host maps
after each run.
**Fix:** route all disk-saving tests through a shared GUID-suffixed
`/Game/PinWrightTests` root + `CleanupTestAsset`-style scope-exit deletion;
audit sequencer/merge/foliage tests for missing cleanup; avoid editor-wide
save-all in tests.

## History
- `#1-initial-repro` `OPEN` reporter — Found 30 leaked .uasset files + a dirty L_Core.umap in the PDS host repo after automation runs dated 2026-07-05/07-10; traced to sequencer, merge_actors, and procedural-foliage tests saving without cleanup (sequencer ones at Content root).
