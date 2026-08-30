---
id: B-asset-save-clobbers-out-of-band-change
title: "asset.save writes the resident in-memory package with no check that the .uasset changed on disk since it was loaded, so a git checkout, an external revert or another tool's write is silently overwritten by the next save of that asset"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, save, in-memory-vs-disk, out-of-band, silent-overwrite, data-loss, stale-package, disk-state-ledger]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# `asset.save` has no idea what is on disk

`asset.save` (`Source/PinWright/Private/Handlers/Asset/AssetSaveHandler.cpp:41-131`) does
three things: `LoadObject` the path, optionally `MarkPackageDirty`, then
`SaveAssetToDiskReportingPresence(Asset, bForce, ...)`. For a package already resident in the
editor, `LoadObject` returns the in-memory object — it does not consult the file — and the
save then writes that in-memory state over whatever the `.uasset` currently holds.

There is no disk-state comparison anywhere on the path. `grep` over the handler for
`GetSavedHash`, `GetTimeStamp`, `FileSize` or any `IFileManager` probe returns nothing. The
handler cannot distinguish these two cases:

1. The resident package is the same revision that is on disk, plus the caller's edits. Saving
   is correct.
2. The `.uasset` on disk was replaced under the editor since the package was loaded or last
   saved — a `git checkout`, a manual revert, a rebase, another tool's write, a teammate's
   sync. The resident package is now a *fork* of an older revision, and saving discards the
   on-disk revision entirely.

Case 2 loses work that the editor never authored and never showed the caller. Nothing in the
response hints at it: `saved: true`, `sizeBytes`, done.

## Provenance

Scoped out deliberately by `B-source-control-revert-no-package-reload` (IN-REVIEW, High) `#6`,
whose fix made `source_control.revert` safe by resynchronising the loaded package through
`USourceControlHelpers::ApplyOperationAndReloadPackages`. Its closing paragraph:

> **Out of scope, needs its own ticket:** `asset.save` still writes the in-memory package
> unconditionally and does not detect that the `.uasset` on disk changed under it since the
> package was loaded/saved, so any OUT-OF-BAND working-tree change (a `git checkout`, an
> external revert, another tool's write) is still silently overwritten by the next save of a
> resident package. Reverting through `source_control.revert` is now safe because it
> resynchronizes; reverting behind the editor's back is not. A fix needs a per-package "disk
> state at last load/save" ledger (mtime+size, or `UPackage::GetSavedHash()` vs the file) that
> does not exist today, and would have to decide refuse-vs-warn — a separate design, not a
> line in this handler.

Re-verified in the current tree: `AssetSaveHandler.cpp` is unchanged in this respect and the
ledger does not exist.

The narrowed hazard after that fix is exactly this: the *in-editor* revert path is now safe,
which makes the *out-of-editor* path the remaining one — and it is the one a developer or a
parallel agent reaches for most often. Every fix host in this workflow does
`git reset --hard` + `git clean -fd` between iterations while an editor may still be resident.

## Why this is not covered by the two nearest tickets

- `E-asset-save-force-no-clean-check` (OPEN, Medium, `blockedBy: [F-asset-save]`) asks for a
  cheap *already-clean* read so callers stop issuing defensive force-saves. That is about
  wasted calls; this is about a write that destroys the file it lands on. Same verb, opposite
  direction — and the readback that ticket asks for would be a natural place to publish this
  one's answer.
- `B-bp-saved-state-corruption-mcp-edits` covers corruption of the Blueprint's own saved state
  by the edit path, not a stale-vs-disk race.

## Severity

**High.** Impact class sits between the rubric's Critical ("a write that ... loses asset
data") and High ("silent false-success ... the caller trusts a result that is a lie"). Taking
High rather than Critical, argued:

- The write itself is not corrupt — it produces a valid, loadable `.uasset`. What is lost is
  the *other* revision, and it is lost silently.
- The loss is recoverable exactly when the discarded revision was tracked, which is the common
  case for the scenario that triggers it (a `git checkout` implies git). It is unrecoverable
  when it is not.
- Reach: `asset.save` runs in nearly every authoring session, which argues a bump up; the
  precondition (an out-of-band write to a package that is still resident) is uncommon, which
  argues a bump down. Recorded as cancelling rather than either applied silently.

Held level with `B-source-control-revert-no-package-reload`, which is High for the same class
in its in-editor form.

**Workaround:** never modify the working tree under a live editor. If you must, run
`asset.reload` (or `source_control.revert`, which now resynchronises) on every touched package
before the next save, or quit the editor first. Do not rely on `asset.save` noticing.

**Fix, and the open design question this ticket exists to settle:** maintain a per-package
disk-state record at load and at each successful save — either `mtime` + size, or
`UPackage::GetSavedHash()` compared against the file — and consult it before writing. Then
decide **refuse vs warn**, which is the part that needs a decision and not just code:

- *Refuse* (`SAVE_DISK_STATE_DIVERGED` + the two states, caller re-reads or forces) is safe by
  default and matches the plugin's refuse-to-write convention on the Blueprint integrity gate
  in this same handler, but breaks any workflow that legitimately expects last-write-wins.
- *Warn* (save, and report `diskStateDiverged: true` with both fingerprints) never blocks, but
  a warning in a response that already says `saved: true` is exactly the shape the board keeps
  filing tickets about.

A `force: true` that already means "bypass the throttle" should probably not silently acquire
"and bypass the divergence gate" — that overloading needs deciding too.

## History
- `#1-out-of-band-write-silently-clobbered` `OPEN` reporter — Source-only; **no editor call, no save, and no working-tree experiment was run this pass** — the tree is mid-verification on another wave and the reproduction (edit an asset, `git checkout` the `.uasset` under the live editor, `asset.save`, inspect the file) was not performed. What is established is the mechanism, read from the whole handler: `AssetSaveHandler.cpp:58` `LoadObject` returns the resident object for an already-loaded package; `:66-69` optionally marks it dirty; `:93` calls `SaveAssetToDiskReportingPresence`; and `grep` over the file for `GetSavedHash`, `GetTimeStamp`, `FileSize` and `IFileManager` returns nothing, so no disk state is consulted at any point. The only gate on the path is the Blueprint integrity check (`:77-83`), which inspects the in-memory graph, not the file. Filed on the explicit instruction of `B-source-control-revert-no-package-reload` `#6`, which scoped this out as needing its own ticket and its own refuse-vs-warn decision; that ticket's own fix narrowed the hazard to the out-of-editor path, which is the one the fix-workflow hosts hit every iteration (`git reset --hard` + `git clean -fd` between runs, per `B-tests-leak-host-content`'s workaround note). Dedup: board greps for `out-of-band`, `git checkout` and `GetSavedHash` surface only that parent, `B-bp-saved-state-corruption-mcp-edits` (different mechanism — corruption by the edit path, not a stale-vs-disk race) and `E-asset-save-force-no-clean-check` (same verb, opposite direction: wasted calls, not a destructive write). Severity High, argued above against Critical; the refuse-vs-warn choice is left open on purpose and is the substance of the fix.
- `#2-savedhash-ledger-refuse-at-the-one-chokepoint` `IN-REVIEW` developer — **Reproduced statically in this checkout before any code was written** (no live editor: the MCP gateway on this project answered `EDITOR_NOT_RUNNING`, and starting one is the orchestrator's job). The chain is unchanged from `#1` and now read end to end: `AssetSaveHandler.cpp:75` `LoadObject` returns the resident object — confirmed against the engine, `StaticLoadObjectInternal` (`UObjectGlobals.cpp:1429-1443`) short-circuits on `StaticFindObjectFast` for an ObjectPath-shaped name and never reaches its `LoadPackage` branch, so the file is not consulted; `:118` calls `SaveAssetToDiskReportingPresence`; `AssetUtils.cpp:775-786` captures `FileSize`/`GetTimeStamp` but only as a POST-save freshness baseline (does this save move the file), never as a comparison against what the package was loaded from; `AssetUtils.cpp:1221` writes via `UEditorAssetLibrary::SaveLoadedAsset`. Plugin-wide, `FPackageFileSummary` had **0** occurrences and `UPackage::GetSavedHash` **0** — the 41 `SavedHash` hits are all `FAssetPackageData::GetPackageSavedHash` inside `AssetDumpCache`, a dump-staleness fingerprint that no save path consults (and unusable here anyway: the registry sets it FROM `Package->GetSavedHash()` for in-memory packages, so it agrees with memory by construction).

    **Ledger: `UPackage::GetSavedHash()` vs the hash in the file's own `FPackageFileSummary`. Not mtime+size.** The deciding question is not cost, it is whether the ledger trips on the editor's own writes. *mtime+size* looks free (`SaveAssetToDiskReportingPresence` already makes both stat calls) but WE would have to own the record — seeded at load, updated after every write — and packages are written from paths this plugin never sees: Ctrl+S in the UI, `editor.save_all`'s `UEditorAssetLibrary::SaveAsset` (`EditorCommandHandler.cpp:264`), `editor.quit`'s `FEditorFileUtils::SaveDirtyPackages`, `FEditorFileUtils::SaveLevel`, the redirector fixup's `PromptForCheckoutAndSave`, engine rename/resave flows. Every unobserved write leaves the record stale and makes the NEXT save a false refusal — the useless-ledger failure the ticket names. *SavedHash* costs one buffered read of the file header (~1 KB, sub-millisecond, against a multi-millisecond package write) and both sides are maintained BY THE ENGINE: `LinkerLoad.cpp:1876` `LinkerRootPackage->SetSavedHash(Summary.GetSavedHash())` on every load, `SavePackage2.cpp:3663` `Package->SetSavedHash(SaveContext.PackageSavedHash)` on every save to a mounted path (gated on `IsUpdatingLoadedPath`, `SavePackageUtilities.cpp:654`), writing the same value into the header it emits. There is no editor save path that can move the file without moving the package's hash with it, so it cannot go stale and there is no state to own. Autosave is the one write that skips the update (`SAVE_FromAutosave` clears `IsUpdatingLoadedPath`) and it writes to `Saved/Autosaves/`, a different file — no false positive. Cooks and PIE duplicates never write the asset's own file. Summary is deserialized through the linker's own `operator<<` rather than seeking `GetSavedHashRelativeOffset` (24 bytes), because that offset is a function of the legacy file version and the operator moves with it.

    **REFUSE, not warn.** A warning attached to a response that already says `saved:true` cannot bring the discarded revision back, and the plugin already refuses to write on the Blueprint integrity gate in this same handler. What makes refusal affordable is the ledger above: because it is engine-maintained on every load and save, the false-positive rate is structurally near zero — with an mtime record I owned, WARN would have been the honest choice. **`force` was deliberately NOT overloaded** (the ticket flagged the risk): the override is a separate `overwriteDiskChanges: boolean` (declared `boolean`, dispatcher type gate satisfied), and a `force:true` save over a diverged file is still refused — asserted by a test.

    **One guard, not N copies.** Write sites counted: **31** callers of `SaveAssetToDiskReportingPresence` + **26** direct callers of `SaveLoadedAssetThrottled` = **57 single-asset write sites sharing exactly one chokepoint** — `AssetUtils.cpp:1221` is the plugin's only `UEditorAssetLibrary::SaveLoadedAsset` call. The guard went there, so all 57 are covered once. Four write sites are NOT covered and are recorded here rather than silently: `editor.save_all` (`UEditorAssetLibrary::SaveAsset` per dirty package, `EditorCommandHandler.cpp:264`), `editor.quit` (`FEditorFileUtils::SaveDirtyPackages`), `McpSafeLevelSave` (`FEditorFileUtils::SaveLevel`, `AssetUtils.cpp:1010`) and `RedirectorFixupPolicy`'s `PromptForCheckoutAndSave` — all bulk/blanket paths with different response contracts; they want their own ticket. `McpSafeAssetSave`'s 153 call sites are not write sites (mark-dirty only) and need nothing.

    **Code.** New `Utils/PackageDiskStateGuard.{h,cpp}` (`PinWrightPackageDiskState`) — `ProbePackageDiskDivergence` measures, `AddPackageDiskStateJson` publishes, `DescribePackageDiskDivergence` writes the refusal line. Shaped after `Handlers/SourceControl/SourceControlPackageResync.{h,cpp}` from the parent ticket: measured post-state struct, honesty flag, Warning-level log carrying values not adjectives. `SaveLoadedAssetThrottled` gained `bAllowDivergedOverwrite` (4th param, defaulted false) and runs the probe after the throttle (a skipped save pays no file read) and after the in-memory integrity gate (cheaper; a corrupt Blueprint keeps reporting the fault it always reported), regardless of the dirty flag (a forced save writes with `bOnlyIfIsDirty=false` and would overwrite from a clean package). New `ESaveLoadedAssetOutcome::RefusedDiskStateDiverged` and `EAssetSaveState::DiskStateDiverged` (wire `diskStateDiverged`) — a first-class state rather than a `Failed`, because nothing was attempted and the remedy is not "clear a fault". `SaveAssetToDiskReportingPresence` forwards the flag and maps the outcome; `IsAssetSaveStateDurable` is unchanged so the `ensureMsgf` postcondition still holds. `asset.save` refuses with `SAVE_DISK_STATE_DIVERGED` + payload; the probe is repeated ONLY on the refusal path, so the happy path pays for one probe, not two.

    **Honesty contract.** `diskState` publishes the MEASURED disk side (`diskSavedHash`, `diskSizeBytes`, `diskModified`, `fileExists`) and names the in-memory side separately (`loadedSavedHash`); when the probe cannot be made it emits `probed:false` + `reason` and **no measured field at all**, so "not compared" can never be read as "they match". Deliberately not refused, and reported as `probed:false`: a package with a zero saved hash (never on disk — every `CreatePackage`+factory create flow, so create-over-existing stays `AssetCreatePolicy`'s decision), an unmounted name, a text-format or unreadable header. A file that is ABSENT is measured (`probed:true, diverged:false, fileExists:false`) and not refused — the resident package is the only surviving copy, so writing it back loses nothing. UE 5.3 has no `UPackage::GetSavedHash` (5.4+; also non-const on 5.4/5.5, hence the non-const `UPackage*`), so the whole probe compiles out there and reports `probed:false` with a reason rather than fabricating a verdict.

    **Error code:** `ERR_SAVE_DISK_STATE_DIVERGED` registered in `Handlers/ErrorCodes.h`, but emitted at the call site as a RAW `TEXT("SAVE_DISK_STATE_DIVERGED")` — `AssetSaveHandler.cpp` has **zero** `ErrorCodes::` references and hand-spells `ASSET_NOT_FOUND`, so citing the constant would flip the file to "adopting" and fail `core.error_codes.RegistryAdoptingFilesUseConstantsOnly` on the other code. Noted at both ends. `Docs/error-code-catalog.md` got the row by hand (+1 code, +1 callsite) with a note that it is not a regeneration.

    **Regression test:** `Tests/Assets/TestAssetSaveDiskStateGuard.cpp`, three ids under `PinWright.assets.AssetSaveDiskGuard.*` (NOT under `PinWright.asset.save`, which is a complete leaf — a suffixed id there would silently swallow it; `Content/Python/check_test_ids.py` re-run after the add: `SCANNED 4818 ids, CLEAN`). Fixture is a `USoundClass` under a GUID-suffixed `/Game/__PW_AssetSaveDiskGuard/<guid>` scratch folder, torn down in `ON_SCOPE_EXIT` along with the parked baseline copy and the folder — never project content. Out-of-band write is a raw `IFileManager::Copy` of a previously saved revision over the live `.uasset`, which is what `git checkout --` does. **Both directions asserted**, as required: `UnchangedFileStillSaves` (no false positive — a fix that refuses everything fails here) and `OutOfBandChangeIsRefused` (no false negative), plus `OverrideOverwritesDeliberately`. Everything drives the `asset.save` RPC rather than the new helper, so on the pre-fix tree the tests COMPILE and go red on assertions rather than failing to build: test 2's three independent reds are `bSuccess` true instead of false, an empty error code, and the `.uasset` bytes replaced by the in-memory revision. Assertions judge FILE BYTES rather than the reported `saved` flag, because two `USoundClass` revisions differing by one float are the same length and the existing freshness verdict is a timestamp/size comparison — a byte assertion cannot be flaky on filesystem timestamp granularity.

    **Not built and not run** (orchestrator owns both). Docs: `Docs/wiki-src/asset.md` `### asset.save` extended with the refusal shape and a worked payload; `Docs/wiki-src/safe-mutation-save.md` gained a `diskStateDiverged` row and a `## Never Change The Working Tree Under A Live Editor` section placed above the file's first `###` (there is none, so it renders). **Residual risks:** (a) the SavedHash covers header+exports up to `PayloadTocOffset`, so a tool that rewrote only a package trailer would not be detected; (b) `AssetHeaderPatcher` (engine rename-by-header-patch) writes a file without going through `UPackage::Save` — it normally operates on unloaded packages, but a resident one patched that way would trip the gate as a true-positive-shaped false positive, and the override is the answer; (c) the four bulk save paths listed above remain unguarded.
