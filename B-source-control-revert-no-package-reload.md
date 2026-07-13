---
id: B-source-control-revert-no-package-reload
title: "source_control.revert reverts the .uasset on disk but never reloads the loaded package — in-memory object stays stale (readbacks lie) and a subsequent asset.save silently un-reverts it"
status: IN-REVIEW
severity: High
category: bug
tags: [source-control, revert, in-memory-vs-disk, stale-package]
encounters: 1
lastSeen: 2026-07-13T11:43:14.0268208+03:00
claimedBy: fuzz2
claimedAt: 2026-07-13T11:50:14.4235341+03:00
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
- `#2-in-review` `IN-REVIEW` developer — GO, severity High confirmed. Independently verified at HEAD: `source_control.revert` (`SourceControlHandler.cpp:356-364`) runs `FRevert` then returns `{success,resultCode,count}` with no package reload, so a loaded `UPackage` stays stale (readbacks lie) and the next save re-persists the pre-revert bytes. Fix approach: extract shared `PinWright::SourceControl::ReloadRevertedPackages` (new `Handlers/SourceControl/SourceControlPackageReload.{h,cpp}`) that resynchronizes each reverted, currently-resident package from the reverted disk bytes via `UPackageTools::ReloadPackages(AssumePositive)` — guarding the active editor level package and revert-of-add (deleted `.uasset`) — and call it from `revert` after a successful `FRevert`; add `reloadedCount` to the response. Regression test drives the shared symbol directly (the handler path is provider-gated / unreachable headless). Compile + differential test verification to follow.
