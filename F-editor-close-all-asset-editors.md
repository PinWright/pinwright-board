---
id: F-editor-close-all-asset-editors
title: "No RPC to enumerate or close ALL open asset-editor tabs (cleanup/observability convenience)"
status: OPEN
severity: Low
category: feature
tags: [editor-quit, asset-editor, cleanup, shutdown, close-all, list-editors, window-heuristic]
encounters: 1
lastSeen: 2026-07-17T13:12:44+03:00
---

# No RPC to enumerate or close all open asset-editor tabs

The `editor` namespace can close editors **for one named asset** (`editor.close_asset`
→ `UAssetEditorSubsystem::CloseAllEditorsForAsset(Asset)`), but there is no RPC to
(a) enumerate every open asset editor or (b) close them all at once. Cleanup and
safe-quit protocols that need "no asset editors open" therefore have no direct verb.

The concrete driver is the `editor.quit` crash documented in
`B-editor-quit-crash-open-asset-editors` — quitting with a Static Mesh Editor tab
open dies in `~FStaticMeshEditor` delegate teardown. The safe protocol is "close all
tabs first", but nothing exposes that. Note that B's own proposed fix is different in
kind: it bakes `CloseAllAssetEditors()` **inside the `editor.quit` handler** as an
automatic mitigation. This ticket asks for a **general-purpose, standalone RPC** —
useful for any cleanup flow, for observability, and for a manual close-inspect-then-quit
protocol — not only for quit. The two are complementary, not duplicates.

## Session evidence (2026-07-17)

Before quitting the user's editor, the agent tried python:
- `unreal.get_editor_subsystem(unreal.AssetEditorSubsystem).close_all_asset_editors()` → `AttributeError` (not exposed in UE 5.7 python)
- `...get_all_edited_assets()` → `AttributeError` too
- `dir()` on the subsystem showed only `close_all_editors_for_asset` and `open_editor_for_assets`

The working fallback was a window-title heuristic: `drive.list_windows` to spot windows
whose title matches an asset name (e.g. "B_SumoPedestal"), then `editor.close_asset {assetPath}`
per asset. That heuristic misses editors **docked inside the main window** (no standalone
window to enumerate) and requires **guessing asset paths from window titles**.

## The C++ capability is already reachable from plugin code

Both halves of the needed subsystem API are already used inside PinWright handlers, so
no new engine access is required — only a thin RPC wrapper:

- **Enumerate**: `UAssetEditorSubsystem::GetAllEditedAssets()` is already called at
  `Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/GraphSelectionHandler.cpp:38`
  (`TArray<UObject*> EditedAssets = AssetEditorSS->GetAllEditedAssets();`).
- **Close (per-asset)**: `CloseAllEditorsForAsset(Asset)` is used in
  `Private/Handlers/Editor/EditorCommandHandler.cpp:496` (the `editor.close_asset` handler),
  `Private/Handlers/Animation/AnimationHandler.cpp:265`, and
  `Private/Handlers/Render/RenderHandler.cpp:271`.
- **Close (all)**: `UAssetEditorSubsystem::CloseAllAssetEditors()` — the all-at-once variant —
  is **not called anywhere** in the plugin (grep: 0 matches). `asset.delete`
  (`Private/Handlers/Asset/AssetManageHandler.cpp:518`) advertises "Closes any open asset
  editors and force-GCs before delete", but the close is a side effect of
  `UEditorAssetLibrary::DeleteAsset` (line 575, force-delete via ObjectTools), not an explicit
  `CloseAllAssetEditors` call — so no existing code path exercises the close-all API.

**Workaround:** `drive.list_windows` → per-asset `editor.close_asset` (fragile: misses docked
editors, needs asset paths guessed from titles).

**Fix:** Add `editor.close_all_asset_editors` (calls
`GEditor->GetEditorSubsystem<UAssetEditorSubsystem>()->CloseAllAssetEditors()`, reports
`closedCount`) and/or `editor.list_open_asset_editors` (returns the asset paths from
`GetAllEditedAssets()`), so quit protocols and cleanup flows stop depending on window-title
heuristics. Model the handler shape on `editor.close_asset` in
`Private/Handlers/Editor/EditorCommandHandler.cpp`.

## History
- `#1-initial-request` `OPEN` reporter — Filed from a live editor-quit-prep session (2026-07-17): needed to close all asset editors before `editor.quit` (per B-editor-quit-crash-open-asset-editors) but no RPC exists. Python `close_all_asset_editors`/`get_all_edited_assets` are AttributeErrors in UE 5.7; the only surface is per-asset `editor.close_asset`, forcing a fragile `drive.list_windows` + per-title heuristic that misses docked editors. The C++ API is already reachable in-plugin (`GetAllEditedAssets` at GraphSelectionHandler.cpp:38; `CloseAllEditorsForAsset` at EditorCommandHandler.cpp:496), so this is a thin-wrapper feature. Distinct from B (which mitigates inside the quit handler) — this asks for a standalone close-all / list RPC.
- `#2-quit-justification-dropped` `OPEN` maintainer — Ruling: open windows preventing a save+quit IS the bug — `B-editor-quit-crash-open-asset-editors` is the real fix (quit must close editors itself / not crash); agents should never need a manual close-all step to quit safely. Severity Medium→Low and title reframed: this RPC remains a standalone cleanup/observability convenience only, no longer justified by the quit path.
