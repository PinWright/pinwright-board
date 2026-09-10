---
id: B-suite-run-dirties-host-repo
title: "A full automation run leaves the host project's git tree dirty: the suite's own editor.save_all test re-saves the shipped startup map, and /Game/PinWrightTests is never swept"
status: OPEN
severity: High
category: bug
tags: [tests, hygiene, host-safety, save-all, teardown, scratch-folder, git]
encounters: 1
lastSeen: 2026-09-10T17:23:34Z
---

# A full automation run leaves the host project's git tree dirty

Running the PinWright automation suite against a real host project mutates tracked,
shipped host content and leaves untracked scratch content behind. After the run the
host's `git status` is untrustworthy, and a test-mutated frontend map is one careless
`git add -A` away from being committed.

Observed on the PDS host (`X:\src\unreal\unreal-fpv-new`, UE 5.8) at the start of the
2026-09-10 session, after earlier test runs:

```
 M Content/System/FrontEnd/Maps/L_Core.umap
 M Content/System/FrontEnd/Maps/L_Core_BuiltData.uasset
?? Content/PinWrightTests/     (368 KB, untracked)
```

Both were reverted by hand. Two independent mechanisms produce them.

## Symptom 1 — the suite performs a real, unfiltered, project-wide save

`FEditorSaveAllRespondsSynchronouslyTest`
(`Source/PinWright/Private/Tests/EditorOps/TestEditorHandlers.cpp:292`) invokes the
production `editor.save_all` handler with an **empty payload**
(`TestEditorHandlers.cpp:310`) purely to assert the response *shape* (savedCount /
totalDirty present, no ticket envelope). `editor.save_all` is an unfiltered flush of
every dirty package in the editor, so on a host project it writes the host's own open
startup map.

Log proof, inside that one test's start/complete window
(`X:\src\unreal\unreal-fpv-new\Saved\Logs\Automation_PinWright_verify2.log`; the run it
records targeted the `unreal-fpv-dev` checkout):

- `:21740` `Test Started. Path={PinWright.editor.save_all.RespondsSynchronously}`
- `:21743` `LogFileHelpers: Saving Map: /Game/System/FrontEnd/Maps/L_Core`
- `:21746` `OBJ SAVEPACKAGE PACKAGE="/Game/System/FrontEnd/Maps/L_Core" FILE=".../Content/System/FrontEnd/Maps/L_Core.umap"`
- `:21752` `LogFileHelpers: Saving Package: /Game/System/FrontEnd/Maps/L_Core_BuiltData`
- `:21799` `Test Completed. Result={Success}`

The save is a side effect, not the assertion: the test never checks `savedCount`'s value.
The map is dirty at that point because tests spawn into the live editor world all run
long; `FScopedEditorWorldActorGuard` (`Tests/TestWorldUtils.h`) restores the dirty flag
only for guarded tests (cf. `B-tests-spawn-live-world-no-guard`,
`B-dirty-world-guard-only-covers-requested-map`).

The sibling `level.save` test does this correctly: every mutation and save is gated on
`IsProbeWorldActive` (`Tests/World/TestLevelSavePathTargeting.cpp:166`) so it can only
ever touch a throwaway probe map, never the host's open map.

## Symptom 2 — `/Game/PinWrightTests` is created but never swept

`/Game/PinWrightTests` is the suite-wide scratch root: **134 open-coded
`/Game/PinWrightTests/` literals across 189 test files**, with no central constant and no
suite-level teardown. Cleanup is strictly per-asset and leaks three ways:

1. **Directories are never removed.** `X:\src\unreal\unreal-fpv-dev\Content\PinWrightTests`
   right now holds **32 directories and 0 files** — per-asset teardown deleted the
   `.uasset` files and left the whole folder tree standing. (Empty dirs are invisible to
   git, which is why that checkout shows nothing; the PDS checkout's 368 KB means files
   survived there too.)
2. **`CleanupTestAsset` gives up silently.** `Tests/TestUtils.h:617`; on a file-delete
   failure (linker still attached / sharing violation) it logs at `Log` level and returns
   (`TestUtils.h:712-717`), leaving the `.uasset` on disk. It also bails before any
   deletion when the file did not exist *at teardown time* — which is exactly the case for
   a package still only dirty in memory, later flushed by symptom 1.
3. **The safe teardown never touches disk.** `PwTestAssetTeardown::DiscardCreatedAssetByObjectPath`
   (`Tests/TestAssetTeardown.h:266`) only routes to `DiscardLoadedAssetNoGc` — rename into
   the transient package + `PackageDeleted`. There is no `IFileManager::Delete`, so its
   ~60 call sites leave any `.uasset` that was actually written to disk.

Deliberate disk-write tests are the obvious feeders — same log, 17 × `Saving Package:
/Game/PinWrightTests/...`, each followed ~2 s later by
`LogAssetRegistry: Warning: ...uasset: package was marked as deleted in editor, but has
been modified on disk. It will once again be returned from AssetRegistry queries.`
(e.g. `:10463` / `:10475`). Sources include
`Tests/Assets/TestAudioCreateSaveWritesToDisk.cpp:39`,
`Tests/Material/TestMaterialCreateSaveWritesToDisk.cpp:40`,
`Tests/Assets/TestMSIRDecompiler.cpp:248`,
`Tests/Assets/TestMetaSoundPatchPreset.cpp:205`,
`PinWrightGeometry/.../TestGeometryConvertNoUVMesh.cpp:115`,
`PinWrightPoseSearch/.../TestPoseSearchCreateSaveWritesToDisk.cpp:203-205`.
Their shared runner `TestCreateHandlerSaveWritesToDisk` (`TestUtils.h:766`) also
**returns before its cleanup** on a handler failure (`TestUtils.h:785`).

There is no global sweep to catch any of this: the only suite-wide hook,
`Tests/AutomationSuiteMaintenance.cpp:395-410` (`OnTestEndEvent`), does GC/memory resets
only.

## Repro

1. Clean host checkout with a startup map (PDS: `/Game/System/FrontEnd/Maps/L_Core`).
2. Run the full PinWright automation suite.
3. `git status` → `L_Core.umap` + `L_Core_BuiltData.uasset` modified, `Content/PinWrightTests/` untracked.

## Relationship to `B-tests-leak-host-content` (IN-REVIEW)

That ticket named the same trigger ("dirty `/Game` packages flushed by the suite's own
editor-wide save-all"), but its shipped fix (`1d41d670`, 2026-07-13) only de-dirtied the
sequencer fixture and cleaned the foliage test's packages, and **explicitly rejected** a
`/Game/PinWrightTests`-level approach. The 2026-08-10 log above is post-fix and still
shows the `L_Core` resave; the PDS host reproduced both symptoms on 2026-09-10. Fixing dirt
sources one fixture at a time is whack-a-mole — this ticket targets the **writer** and the
**missing suite-level sweep** instead.

**Workaround:** after every run,
`git checkout -- Content/System/FrontEnd/Maps/L_Core.umap Content/System/FrontEnd/Maps/L_Core_BuiltData.uasset`
and `rm -rf Content/PinWrightTests`. (The fuzz hosts hide this behind a per-iteration
`git reset --hard` + `git clean -fd`; a manual dev checkout like PDS does not.)

**Fix:**
1. `TestEditorHandlers.cpp:292` must stop performing an unfiltered project-wide save on a
   host project. Assert the JSON shape against `EditorSaveAllDiagnostic::BuildSaveAllResultJson`
   directly, or gate the invocation on a throwaway probe world the way
   `TestLevelSavePathTargeting.cpp:166` does. Hard rule to enforce suite-wide: **a test may
   never save a shipped host map.**
2. Put the scratch root behind one constant and add a suite-level sweep to
   `AutomationSuiteMaintenance.cpp` (it already owns the framework hook at `:395-410`):
   on suite end, recursively delete `<Project>/Content/PinWrightTests` (files *and*
   directories) and notify the asset registry.
3. Make `DiscardCreatedAssetByObjectPath` (`TestAssetTeardown.h:266`) delete the on-disk
   `.uasset`, or refuse saved fixtures, so its ~60 call sites cannot leave a file.
4. Optional ratchet: have the maintenance hook fail the run when a test leaves a dirty
   package outside `/Game/PinWrightTests`, so a new leak source is caught at its source
   test instead of at the next `git status`.

## History
- `#1-initial-repro` `OPEN` reporter — PDS host (unreal-fpv-new, UE 5.8) showed `M Content/System/FrontEnd/Maps/L_Core.umap`, `M .../L_Core_BuiltData.uasset` and `?? Content/PinWrightTests/` (368 KB) after automation runs; reverted by hand. Traced to two mechanisms: `PinWright.editor.save_all.RespondsSynchronously` (TestEditorHandlers.cpp:292/:310) invoking the real unfiltered `editor.save_all`, log-proven writing `L_Core.umap` + `L_Core_BuiltData.uasset` inside its own start/complete window (Automation_PinWright_verify2.log:21740-21799); and `/Game/PinWrightTests` having no suite-level sweep (32 empty dirs still standing in the -dev checkout; `CleanupTestAsset` logs-and-returns on delete failure, TestUtils.h:712-717; `DiscardCreatedAssetByObjectPath`, TestAssetTeardown.h:266, never deletes the on-disk file). Complements B-tests-leak-host-content, whose per-fixture fix (1d41d670) did not stop the L_Core resave.
