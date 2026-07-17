---
id: B-editor-quit-save-true-fails-where-asset-save-works
title: "editor.quit save:true bulk-saves via FEditorFileUtils::SaveDirtyPackages and fails where per-asset asset.save succeeds"
status: OPEN
severity: Medium
category: bug
tags: [editor, editor-quit, save, save-dirty-packages, asymmetry]
---

# editor.quit save:true bulk save fails on packages that per-asset asset.save saves fine

`editor.quit` with `save:true` runs a single **bulk** save via
`FEditorFileUtils::SaveDirtyPackages(bPromptUserToSave=false, bSaveMapPackages=true, bSaveContentPackages=true)`
(`Handlers/Editor/EditorQuitHandler.cpp:130-133`), trusts that call's bare bool
return, and on `false` emits `SAVE_FAILED` and refuses to exit
(`EditorQuitHandler.cpp:134-142`). Reported behaviour (carried from a prior
session's handoff, not freshly reproduced this session): the bulk path left the
target packages dirty / refused the quit, yet calling per-asset `asset.save` on
those same packages saved them successfully, after which `editor.quit` proceeded
cleanly. Exact error text was not preserved in the handoff, so treat the repro
as approximate — but the asymmetry (bulk save path fails, per-asset path works)
is the reliable claim.

## Why the asymmetry is plausible at the source level

The two save paths use **different engine APIs**:

- **editor.quit** → `FEditorFileUtils::SaveDirtyPackages(...)` — the bulk
  save-all-dirty UnrealEd entry point (`EditorQuitHandler.cpp:130`). It returns a
  single aggregate bool that can go `false` for a whole class of reasons
  (a package that needs a source-control checkout the unattended path suppresses,
  a read-only file, a PIE-locked asset, a package the save-dialog machinery can't
  resolve). editor.quit trusts that bool with no per-package attribution, no
  integrity gate, and no on-disk-presence verification.
- **asset.save** → `SaveAssetToDiskReportingPresence` (`Handlers/Asset/AssetSaveHandler.cpp:90`)
  → `SaveLoadedAssetThrottled` → `UEditorAssetLibrary::SaveLoadedAsset(Asset, /*bOnlyIfIsDirty=*/!bForce)`
  (`Utils/AssetUtils.cpp:712`), a **direct per-package** write, then verifies the
  `.uasset` actually landed on disk (`AssetUtils.cpp:481-490`). The per-package
  write is more permissive than the bulk save-all dialog path.

Strong corroboration inside this same codebase: **`editor.save_all` was already
migrated off the bulk API for exactly this reason.** `SaveDirtyPackagesWithIntegrityGate`
(`Handlers/Editor/EditorCommandHandler.cpp:234-281`) no longer calls
`FEditorFileUtils::SaveDirtyPackages`; it loops the dirty set and saves each
package individually via `UEditorAssetLibrary::SaveAsset(PackagePath, false)`
(`EditorCommandHandler.cpp:261`), adding an integrity gate and PIE-aware
per-asset failure reasons (`B-editor-save-all-pie-diagnostic`). `editor.quit` is
the sibling handler left behind on the abandoned bulk mechanic — so the reported
"bulk fails where per-asset works" is precisely the failure mode that drove the
save_all rewrite.

Distinct from `B-editor-quit-unsaved-changes-unregistered` (that ticket is about
the missing `ERR_UNSAVED_CHANGES` registry constant on the refusal path, not the
save mechanics) and from the `B-editor-save-all-*` family (those cover
`editor.save_all`, which already has the per-package path; `editor.quit` does not).

**Workaround:** call `asset.save` per dirty package first, then `editor.quit`
(with no `save`, or `save:true` on the now-clean set).

**Fix:** route editor.quit's `save:true` branch through the same
`EditorSaveAllDiagnostic::SaveDirtyPackagesWithIntegrityGate` helper
`editor.save_all` already uses (per-package `UEditorAssetLibrary::SaveAsset`
loop + integrity gate + per-asset failure attribution), or through the same
`asset.save` mechanics — instead of the bulk `FEditorFileUtils::SaveDirtyPackages`.
That also gives editor.quit the per-package failure detail it currently lacks
when it reports `SAVE_FAILED`.

## History
- `#1-initial-report-from-handoff` `OPEN` reporter — Repro carried from the immediately-preceding session in this work-stream via its handoff (recorded there as a known-unticketed plugin bug); NOT freshly reproduced — this session used only clean-quit paths (dirtyCount 0). Reported: `editor.quit save:true` failed to save dirty packages (quit refused / packages stayed dirty) while per-asset `asset.save` on the same packages succeeded, after which quit proceeded cleanly; exact error text not preserved in the handoff. Source review confirms a real API divergence that makes the asymmetry plausible: editor.quit bulk-saves via `FEditorFileUtils::SaveDirtyPackages` (`EditorQuitHandler.cpp:130-133`) and trusts its aggregate bool, whereas `asset.save` writes per-package via `UEditorAssetLibrary::SaveLoadedAsset` with disk-presence verification (`AssetSaveHandler.cpp:90` → `AssetUtils.cpp:712`, `481-490`). Corroborated by `editor.save_all` already having abandoned the bulk API for a per-package `UEditorAssetLibrary::SaveAsset` loop (`EditorCommandHandler.cpp:234-281`, save at `:261`).
