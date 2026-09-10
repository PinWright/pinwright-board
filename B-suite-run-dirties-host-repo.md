---
id: B-suite-run-dirties-host-repo
title: "A full automation run leaves the host project's git tree dirty: the suite's own editor.save_all test re-saves the shipped startup map, and /Game/PinWrightTests is never swept"
status: IN-REVIEW
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
- `#2-writer-guard-and-suite-sweep` `IN-REVIEW` developer — Both halves fixed structurally, no host content touched by hand. (1) WRITER: `Source/PinWright/Private/Tests/AutomationSuiteMaintenance.{h,cpp}` gains `FScopedForeignDirtyPackageSuspension`, which walks the same `FEditorFileUtils::GetDirtyWorldPackages` + `GetDirtyContentPackages` pair `editor.save_all` walks, clears the dirty flag of every dirty package NOT under the scratch root, and restores it on scope exit (names, not pointers, so a collected package is skipped rather than dereferenced). `Tests/EditorOps/TestEditorHandlers.cpp:292` now wraps its `editor.save_all` invocation in that guard, so the handler still runs for real and every shape assertion still measures the real response, but the dirty set it flushes is fixture-only — a shipped host map can no longer be persisted, and an unsaved host edit is neither written nor discarded. `editor.save_all` is `RPC_NO_PARAMS` (EditorCommandHandler.cpp:458): there is no dryRun and no package filter to verify against, which is why the fix is on the dirty set rather than on the call. (2) SWEEP: same file gains `ScratchRootPackagePath()` (the one spelling of `/Game/PinWrightTests`), `ScratchRootContentDir()`, `IsScratchPackageName()`, `ScratchRootExistsOnDisk()` and `SweepScratchRoot()`, which routes every `.uasset`/`.umap` under the root through the existing `CleanupTestAsset` (detach + delete + unregister, never `ObjectTools::ForceDeleteObjects`), plain-deletes any other leftover file, reports the ones it could not remove, and then removes the emptied directories deepest-first — the "32 directories, 0 files" residue per-asset teardown leaves behind. Regression tests in new `Tests/Infra/TestSuiteScratchRootHygiene.cpp`: `PinWright.infra.contract.SuiteMaintenance.ForeignDirtyPackagesSuspendedAndRestored` (counterfactual: revert the ctor body of `FScopedForeignDirtyPackageSuspension` and the `/Game` probe package stays dirty inside the guard scope, so "a dirty package outside the scratch root is suspended" fails); `PinWright.infra.contract.SuiteMaintenance.ScratchRootSweepRemovesSavedFixtureFile` (counterfactual: revert the file-deletion loop in `SweepScratchRoot` and the saved probe `.uasset` is still `FileSize >= 0` after the sweep, so "the scratch probe .uasset is gone from disk" fails; revert the directory pass and the probe's own GUID subfolder still exists, so the emptied-directory assertion fails); and the suite-end gate `PinWright.zz_suite_end.ScratchRootIsEmptyOnDisk`, which runs the sweep last (the `zz_` first segment is what orders it after every other `PinWright.*` id — the controller sorts by display name at AutomationControllerManager.cpp:1047) and fails naming any file that survived, emitting `PINWRIGHT_ASSERTIONS_SKIPPED` only when the root was never created. Docs: `Docs/test-organization.md` → Fixture Teardown gains "The scratch root, and the two rules that keep the host repo clean"; one paragraph added to the plugin CLAUDE.md Testing section. NOT done, deliberately: the 134 open-coded `/Game/PinWrightTests/` literals were left alone (the constant is used by the new code only — rewriting 189 files is not a surgical diff); `DiscardCreatedAssetByObjectPath` was left as-is (fix item 3) since the suite-level sweep now catches what it leaves; per-test world-guard coverage stays with `B-tests-spawn-live-world-no-guard`, and the writer guard makes it non-load-bearing for host-repo cleanliness; the optional ratchet (fix item 4, fail the run on a dirty non-scratch package) was not added — the suite-end gate covers the disk half only. Limitation: a run SCOPED to a filter that excludes `PinWright.zz_suite_end.*` does not sweep; no engine-pre-exit hook was added because `CleanupTestAsset` reaches `IAssetRegistry::GetChecked()`, which is not safe to assume alive at shutdown.
- `#3-verifier-fixes-and-a-gap-outside-the-swept-root` `IN-REVIEW` developer — Two verifier findings fixed. (a) The `.umap` branch of `SweepScratchRoot` did not do what its comment and the doc claimed: `CleanupTestAsset` resolves filenames with `GetAssetPackageExtension()` only (`Tests/TestUtils.h:636-638`), so for a map it probes a `<pkg>.uasset` that does not exist, skips `ResetLoaders` and returns before deleting — while still having renamed the world into `/Transient`, so a post-hoc `FindPackage` cannot reach the original package either. `ResetLoaders(FindPackage(nullptr, PackageName))` now runs on the map package BEFORE `CleanupTestAsset`, so the fallback `IFileManager::Delete` no longer runs against an attached map linker (`ERROR_SHARING_VIOLATION`); the `Docs/test-organization.md` sentence was corrected to describe the real two-step rather than a uniform `.uasset`/`.umap` path. (b) `SweepScratchRoot` gained an optional `SubDirectory` argument (rejected unless it resolves under the root) and `PinWright.infra.contract.SuiteMaintenance.ScratchRootSweepRemovesSavedFixtureFile` now passes its own GUID probe folder: that test runs mid-suite — `PinWright.infra.*` is well after `PinWright.editor.*`, and the sprint-c5 log shows eight scratch `.uasset`s already on disk by then — so the global sweep it used to run was detaching and deleting other tests' live fixtures as a side effect. Its count assertion tightened from `>= 1` to `== 1`, which is the assertion that proves the scoping. Only `PinWright.zz_suite_end.ScratchRootIsEmptyOnDisk` sweeps the whole root, and both the file header and the doc now say so. GAP, documented not enforced: `Tests/World/TestLevelSavePathTargeting.cpp:76` builds its probe maps at `/Game/Maps/__EARG_LevelSaveProbe_<GUID>`, i.e. straight into the host's own `Content/Maps/`, outside the swept root. It has its own careful discard protocol (`:88` onward, deliberately not routed through force-delete), but if that protocol fails the suite-end gate cannot see it and the `.umap`/`.uasset` lands in the host's `git status`. Any fixture written outside `/Game/PinWrightTests` is invisible to the gate by construction; re-homing that probe, or extending the gate to a second declared root, is a follow-up and is not done here.
