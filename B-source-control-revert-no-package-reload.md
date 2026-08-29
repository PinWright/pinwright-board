---
id: B-source-control-revert-no-package-reload
title: "source_control.revert reverts the .uasset on disk but never reloads the loaded package — in-memory object stays stale (readbacks lie) and a subsequent asset.save silently un-reverts it"
status: IN-REVIEW
severity: High
category: bug
tags: [source-control, revert, in-memory-vs-disk, stale-package]
encounters: 1
lastSeen: 2026-07-13T11:43:14.0268208+03:00
---

# `source_control.revert` leaves the in-memory package unsynchronized with the reverted disk state

## What's wrong

`source_control.revert` executes the raw provider revert (`FRevert`) against the
on-disk `.uasset` files but does **not** unload/reload the corresponding loaded
`UPackage`. So after a successful revert of a currently-loaded asset:

1. The disk/git state is correctly restored to the committed baseline
   (`source_control.status` reports `isModified:false`), **but**
2. The in-memory `UObject` still holds the pre-revert (modified) values, so every
   in-editor readback (`material.authoring.get_material_info`, and by the same
   mechanism any other reflect-the-live-object read) returns the **stale,
   un-reverted** value — the caller is told the file is clean by `status` yet
   reads the modified content back. The caller trusts a lie.
3. Worse: because the package is left holding the pre-revert content, any
   subsequent save of that package (`asset.save force:true`, `editor.save_all`,
   an autosave, or the editor's shutdown save-prompt) writes the stale in-memory
   state **back to disk**, silently **undoing the revert**. `source_control.status`
   flips back to `isModified:true`.

The editor's own Content-Browser revert flow avoids this by unloading/reloading
the reverted packages (`FSourceControlWindows::RevertFiles` ->
`UPackageTools::ReloadPackages`). This handler skips that step, so the "discard my
local changes and go back to the committed baseline" task is only half-completed:
the bytes on disk are reverted, but the live editor is left in an inconsistent
state that both misreports the asset and re-corrupts it on the next save.

Note a confounder observed during confirmation: the staleness eventually
self-heals — the editor's external-file-change watcher reloads the reverted
package after some delay (an earlier revert from the same session read back the
correct reverted value once enough time had passed). But there is a real window
(seconds and up, and it does not resolve on a `source_control.status` refresh)
during which readbacks lie and a save persists the un-reverted state. The defect
is that `revert` does not make the in-memory package consistent synchronously.

## Guilty source line

`Plugins/PinWright/Source/PinWright/Private/Handlers/SourceControl/SourceControlHandler.cpp:356`

```cpp
    const ECommandResult::Type ExecResult =
        Provider->Execute(ISourceControlOperation::Create<FRevert>(), Filenames);
```

The handler returns immediately after the provider `Execute` — there is no
`UPackageTools::ReloadPackages` / package unload+reload of the reverted files
anywhere in the `source_control.revert` handler (lines 340-365). The reload
mechanism already exists in the plugin: `asset.reload` in
`Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp`
calls `UPackageTools::ReloadPackages` (see `F-asset-reload-from-disk`), so revert
can reuse the same path for the packages it just reverted.

## What it should do

After a successful `FRevert` on a set of filenames, reload the affected loaded
packages from the (now reverted) disk bytes — e.g. resolve each reverted filename
to its loaded `UPackage` and run `UPackageTools::ReloadPackages(...)` (guarding
the active level package as `asset.reload` does) — so that:
- readbacks reflect the reverted content immediately, and
- a subsequent save cannot re-persist the pre-revert state.

Optionally report per-file `reloaded`/`wasLoaded` in the response so the caller
knows the in-memory state was resynchronized.

## Verbatim repro (replayed at HEAD via mcp__pinwright__call, Git provider)

Target: `/Game/Global/DemoRoom/Materials/M_Glow` (a base UMaterial, default
`twoSided:false`).

1. `material.authoring.set_two_sided` `{"assetPath":".../M_Glow","twoSided":true}`
   -> `{"assetPath":".../M_Glow","twoSided":true}`
2. `asset.save` `{"assetPath":".../M_Glow"}`
   -> `{"saved":true,"sizeBytes":10404}`
3. `material.authoring.get_material_info` `{"assetPath":".../M_Glow"}`
   -> `"twoSided":true` (modify took in memory);
   `source_control.status` -> `"isModified":true,"isUnchanged":false` (git dirty).
4. `source_control.revert` `{"assetPaths":[".../M_Glow"]}`
   -> `{"success":true,"resultCode":1,"count":1}`
5. `material.authoring.get_material_info` `{"assetPath":".../M_Glow"}` (no intervening status)
   -> `"twoSided":true`  <-- STALE: disk was reverted, in-memory still modified.
6. `source_control.status` `{"assetPaths":[".../M_Glow"]}`
   -> `"isModified":false,"isUnchanged":true` (disk/git IS reverted/clean).
7. `material.authoring.get_material_info` `{"assetPath":".../M_Glow"}` (after the status refresh)
   -> `"twoSided":true`  <-- STILL STALE: a status refresh does not reload the package.
8. `asset.save` `{"assetPath":".../M_Glow","force":true}`
   -> `{"saved":true,"sizeBytes":10404}` (writes the stale in-memory value back)
9. `source_control.status` `{"assetPaths":[".../M_Glow"]}`
   -> `"isModified":true,"isUnchanged":false`  <-- the revert was SILENTLY UNDONE.

## Severity

severity rationale: impact=silent-wrong-data-and-silent-undo (readbacks return
the un-reverted value while `status` says clean, and the next save silently
re-persists the pre-revert content, reversing the user's explicit "discard my
changes") x reach=source-control revert core verb -> High. Not Critical: the
asset stays structurally valid (no corruption/crash) and the un-revert requires a
follow-on save; but it is materially worse than a self-healing readback because
the revert verb silently fails its own contract.

## History
- `#1-initial-repro` `OPEN` reporter — Realism task ("experiment on DemoRoom materials, then discard local edits on all but one via in-editor Git source control"). Replay-confirmed at HEAD on `/Game/Global/DemoRoom/Materials/M_Glow`: after `set_two_sided:true` + `asset.save` + `source_control.revert`, `source_control.status` reports the file clean (`isModified:false`) yet `material.authoring.get_material_info` still reads `twoSided:true` (stale in-memory package), and a follow-up `asset.save force:true` re-persists the stale value, flipping `status` back to `isModified:true` — silently undoing the revert. Root cause: `SourceControlHandler.cpp:356` runs `FRevert` with no package unload/reload (the editor's own revert flow reloads via `UPackageTools::ReloadPackages`; `asset.reload` already wraps that path in `AssetManageHandler.cpp`). The attempt's self_report framed this as "not a revert bug, just the editor not hot-reloading" and downplayed it; the replay shows it is a revert-verb defect with a data-integrity consequence (silent un-revert on next save). Related: `F-asset-reload-from-disk` (the reload capability the fix should reuse).
- `#2-in-review` `IN-REVIEW` developer — GO, severity High confirmed. Independently verified at HEAD: `source_control.revert` (`SourceControlHandler.cpp:356-364`) runs `FRevert` then returns `{success,resultCode,count}` with no package reload, so a loaded `UPackage` stays stale (readbacks lie) and the next save re-persists the pre-revert bytes. Fix approach: extract shared `PinWright::SourceControl::ReloadRevertedPackages` (new `Handlers/SourceControl/SourceControlPackageReload.{h,cpp}`) that resynchronizes each reverted, currently-resident package from the reverted disk bytes via `UPackageTools::ReloadPackages(AssumePositive)` — guarding the active editor level package and revert-of-add (deleted `.uasset`) — and call it from `revert` after a successful `FRevert`; add `reloadedCount` to the response. Regression test drives the shared symbol directly (the handler path is provider-gated / unreachable headless). Shipped: `SourceControlHandler.cpp` now calls `ReloadRevertedPackages(Filenames)` after a successful `FRevert` and returns `reloadedCount`; new `SourceControlPackageReload.{h,cpp}`; new test `Tests/SourceControl/TestSourceControlRevertReload.cpp` = `PinWright.source_control.RevertReloadsInMemoryPackage` (saves a SoundClass at Volume=0.5, mutates the resident copy to 0.123 without saving, calls the helper, asserts the reload restores 0.5 and reloadedCount==1). Verified: plugin compiles clean; the new test passes (Result={Success}); differential-confirmed — pre-fix the test TU does not compile without the introduced helper symbol.
- `#3-attempt-abandoned` `OPEN` supervisor — attempt-failed: the fix workflow was deliberately stopped by the user mid-pipeline (after implement + local verification, before the Commit phase pushed to origin). The verified diff was left uncommitted in fuzz2's plugin clone and will be swept by the next run's Baseline; nothing reached origin. Reopened and lease released for a fresh attempt — the `#2` analysis and fix approach remain valid as a starting point.
- `#4-additional-engine-helper-review` `OPEN` reporter — Additional evidence: **Adversarial review A — current defect remains; the abandoned fix needs an engine-backed implementation.** Actuality: CONFIRMED CURRENT. Framing: title and High severity remain accurate: `source_control.revert` still calls synchronous `FRevert` and immediately responds, with no package synchronization; the current source-control tests cover provider/status/connect only, not revert/reload. Proposed fix: **INCOMPLETE**, a custom per-file `UPackageTools::ReloadPackages` helper does not reproduce UE's source-control edge-case handling for active/world or external packages, deleted/reintroduced packages, file-deletion notifications, and status recache. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\SourceControl\SourceControlHandler.cpp:356-364` `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetManageHandler.cpp:1374-1401` `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\SourceControl\TestSourceControlHandlers.cpp:19-283` `C:\UE_5.8\Engine\Source\Editor\SourceControlWindows\Private\SSourceControlRevert.cpp:679-684` `C:\UE_5.8\Engine\Source\Developer\SourceControl\Private\SourceControlHelpers.cpp:977-1075,1197-1304` `X:\src\unreal\.pinwright-board\B-source-control-revert-no-package-reload.md:115`. Runtime: NOT VERIFIED. Recommendation: KEEP; replace the raw provider call with `SourceControlHelpers::RevertAndReloadPackages(Filenames, false, false)` or its `ApplyOperationAndReloadPackages` equivalent, then add a loaded-package mutate/revert/read/save regression and provider-gated runtime replay.
- `#5-additional-interactive-helper-caveat` `OPEN` reporter — Additional evidence: **Adversarial review B — A's core verdict survives, but its proposed UE-helper substitution is incomplete.** Actuality: CONFIRMED CURRENT. Framing: title and High severity remain accurate: the handler still performs synchronous-default `FRevert`, sends its response immediately, and has no revert/reload test. I agree with A that the Content Browser uses `SourceControlHelpers::RevertAndReloadPackages`, but disagree that calling it as `RevertAndReloadPackages(Filenames, false, false)` alone is sufficient for this RPC: that wrapper hardcodes interactive reload, while direct automation can leave packages dirty and trigger a modal prompt/block. Use the lower-level helper with `bInteractive=false` (or an equivalent noninteractive engine-backed adapter). Also, `ApplyOperationAndReloadPackages` returns the provider operation result while reload failures are only logged, so the RPC must surface reload failure/unsynchronized paths instead of reporting a clean revert. Proposed fix: **INCOMPLETE**, preserve UE's deletion notifications, world/external filtering, unload/reload, and status recache, but make interaction and result handling explicit. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\SourceControl\SourceControlHandler.cpp:356-364` `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\SourceControl\TestSourceControlHandlers.cpp:19-283` `C:\UE_5.8\Engine\Source\Developer\SourceControl\Public\SourceControlHelpers.h:419-438` `C:\UE_5.8\Engine\Source\Developer\SourceControl\Private\SourceControlHelpers.cpp:1192-1304,1467-1515` `C:\UE_5.8\Engine\Source\Editor\SourceControlWindows\Private\SSourceControlRevert.cpp:657-684`. Runtime: NOT VERIFIED. Recommendation: KEEP; implement the noninteractive/error-aware engine-backed path, then provider-gated mutate/revert/read/save coverage including deleted/reintroduced and world/external cases.
- `#6-engine-backed-resync-shipped` `IN-REVIEW` developer — Defect re-confirmed in THIS checkout (`X:\src\unreal\unreal-fpv-new`, not the `-dev` clone the prior reviews cited): `SourceControlHandler.cpp:356-357` still ran a bare `Provider->Execute(FRevert)` and returned `{success,resultCode,count}` with no package synchronization; the file was clean of any prior fix. Implemented review B's recommendation, not review A's: new `Handlers/SourceControl/SourceControlPackageResync.{h,cpp}` (`PinWrightSourceControlResync::ApplyAndResyncPackages`) runs the caller's operation through the engine's own `USourceControlHelpers::ApplyOperationAndReloadPackages(Filenames, Op, bReloadWorld=false, bInteractive=false)` — the same path `SSourceControlRevert.cpp:679` reaches via `RevertAndReloadPackages`, so loader unlink, world/non-world split, revert-of-add delete+unload and the status recache all come from the engine. `RevertAndReloadPackages` itself is deliberately NOT used: it leaves `bInteractive` at its default `true`, which reaches `UPackageTools::ReloadPackages` in `Interactive` mode and opens a modal dirty-package dialog (`PackageTools.cpp:718-746`) — a hang on a headless RPC host; `bInteractive=false` maps to `AssumePositive` (`SourceControlHelpers.cpp:1249`), i.e. disk wins with no prompt, matching `asset.reload`. Honesty per the response convention: the helper MEASURES the post-state rather than trusting the engine's return value (which is the OPERATION's result — reload failures are only logged), comparing a pre-call `TWeakObjectPtr<UPackage>` against `FindPackage` afterwards, and `source_control.revert` now returns measured `loadedCount`/`reloadedCount`/`removedCount`/`staleCount`, `resynchronized`, per-path `files[]` (`wasLoaded`/`reloaded`/`removed`/`stale`) alongside the REQUESTED `count`, plus a `resyncWarning` only when `staleCount>0`. Loaded map packages and loaded external packages of a loaded world are REFUSED up front with `REVERT_REQUIRES_UNLOADED_PACKAGE` + `blockedPackages` and nothing is reverted (the engine aborts that batch too, but only with a log line); `ERR_REVERT_REQUIRES_UNLOADED_PACKAGE` added to `Handlers/ErrorCodes.h` for the registry contract while the handler keeps its non-adopting raw-literal style. Reload side effects established and documented: open asset editors are closed and reopened on the new object (`AssetEditorSubsystem.cpp:1468-1540`), editor selection sets are cleared (`PackageTools.cpp:938`), every `UObject*` into a reloaded package dies; the plugin itself caches no `UObject*` across an RPC (audited — all `TStrongObjectPtr`/`AddToRoot` sites are single-operation scoped), so there is no plugin-internal dangling surface. Regression test `Tests/SourceControl/TestSourceControlRevertResync.cpp` = `PinWright.source_control.RevertResyncReloadsInMemoryPackage` + `PinWright.source_control.RevertResyncSurvivesFollowUpSave`: saves a `USoundClass` at Volume=0.25, copies the bytes aside as the committed baseline, saves Volume=0.875 as the local change, then applies a baseline-restore operation (the on-disk half of `FRevert`, so no provider is needed — the handler is provider-gated and unreachable headless) and asserts (a) the resident object now reads 0.25 and `reloadedCount==1`/`staleCount==0`, and (b) after a follow-up `SavePackage` the bytes on disk STILL read 0.25 when force-reloaded. Both fail pre-fix. `check_test_ids.py`: CLEAN, 4776 unique ids. Wiki `Docs/wiki-src/source_control.md` documents the measured fields, the refusal, and the reload side effects. NOT compiled/run here — orchestrator builds. **Out of scope, needs its own ticket:** `asset.save` still writes the in-memory package unconditionally and does not detect that the `.uasset` on disk changed under it since the package was loaded/saved, so any OUT-OF-BAND working-tree change (a `git checkout`, an external revert, another tool's write) is still silently overwritten by the next save of a resident package. Reverting through `source_control.revert` is now safe because it resynchronizes; reverting behind the editor's back is not. A fix needs a per-package "disk state at last load/save" ledger (mtime+size, or `UPackage::GetSavedHash()` vs the file) that does not exist today, and would have to decide refuse-vs-warn — a separate design, not a line in this handler.
- `#7-asset-save-residue-now-filed` `IN-REVIEW` reporter — **Status deliberately NOT changed; no code touched.** The out-of-scope item `#6` named ("`asset.save` still writes the in-memory package unconditionally and does not detect that the `.uasset` on disk changed under it [...] needs a per-package disk-state ledger [...] and would have to decide refuse-vs-warn — a separate design, not a line in this handler") is now **`B-asset-save-clobbers-out-of-band-change`** (OPEN, High). Re-verified against the current working tree: `Handlers/Asset/AssetSaveHandler.cpp` reaches `SaveAssetToDiskReportingPresence` at `:93` after a `LoadObject` at `:58` that returns the resident object for an already-loaded package, and `grep` over the file for `GetSavedHash`, `GetTimeStamp`, `FileSize` and `IFileManager` returns nothing — no disk state is consulted anywhere on the path, and the only gate present (the Blueprint integrity check at `:77-83`) inspects the in-memory graph rather than the file. The new ticket carries the refuse-vs-warn decision `#6` left open as its substance, including the sub-question of whether `force:true` (today "bypass the throttle") should also mean "bypass the divergence gate". **Scope for this ticket's tester is unchanged and is `#6`'s revert resynchronisation only.** Worth noting for whoever picks the child: `#6`'s fix narrowed the hazard to the *out-of-editor* path, which is the one the fix-workflow hosts hit every iteration — they `git reset --hard` + `git clean -fd` between runs while an editor may still be resident.
