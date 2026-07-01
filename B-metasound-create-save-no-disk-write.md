---
id: B-metasound-create-save-no-disk-write
title: "create_metasound_patch / create_metasound_preset save:true never writes the .uasset — McpSafeAssetSave only marks dirty, but existsAfter:true implies persistence"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, metasound, save, create-metasound-patch, mcp-safe-asset-save, silent-failure, false-success]
---

# create_metasound_patch's save:true claims the asset is saved but writes nothing to disk

`audio.authoring.create_metasound_patch` (and the sibling
`create_metasound_preset`) take a `save` param defaulting to `true` and,
on success, return an `existsAfter:true` verification block — so a caller
reasonably believes the new `.uasset` is on disk. It is not. With
`save:true` the handler only **marks the package dirty**; the file is not
written until a separate `editor.save_all`. After the editor closes (or a
fuzz `git reset --hard`) the asset is gone, with no error ever surfaced.

This is the MetaSound-create analog of `B-niagara-save-no-disk-write`
(niagara path) and `B-create-level-saved-true-no-umap` (level path): a
`save:true`-style success signal backed only by a mark-dirty, masked by a
registry-based `existsAfter:true`.

## Root cause (verified in source)

`MetaSoundPatchPresetHandler.cpp` routes the `save` param through the
shared no-op helper:

```cpp
// create_metasound_patch
bool bSave = Ctx.GetBool(TEXT("save"), true);
...
if (bSave) { McpSafeAssetSave(Patch); }            // :90-93
AddAssetVerification(Result, Patch);               // sets existsAfter:true (registry, not disk)

// create_metasound_preset
if (bSave) { McpSafeAssetSave(NewAsset); }          // :217-220
```

`McpSafeAssetSave` (`Utils/AssetUtils.cpp:214-226`, quoted in
`B-niagara-save-no-disk-write`) **never calls any package-save API** —
it only `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` and
returns `true` unconditionally (the deferred-save behaviour is deliberate
and corruption-driven, per `B-bp-saved-state-corruption-mcp-edits`). So
the patch/preset is registered (hence `existsAfter:true`,
`describe_metasound` reads it) but never lands on disk. The handler's
`save:true` default + `existsAfter:true` response together imply a
persistence that did not happen, and the response carries no
`pendingFlush`/disk-presence signal to say otherwise.

`B-niagara-save-no-disk-write` explicitly names this same path as affected
("MetaSound, StateTree, etc. — all report a save that did not happen") but
**scopes its fix to the niagara save path only**, leaving the MetaSound
create handlers unfixed — there is no MetaSound save ticket. This ticket
fills that gap.

## What it should do

Mirror the accepted sibling fixes (`B-niagara-save-no-disk-write` #2,
`B-create-level-saved-true-no-umap`): persist for real on the MetaSound
create path (route `save:true` through the in-tree real-save helper
`SaveLoadedAssetThrottled` instead of `McpSafeAssetSave`), then probe
on-disk presence (`IFileManager::FileSize(PackageFilename)`) and gate the
result via the shared `ShouldTreatAssetSaveAsSuccess` predicate — report a
`saved`/disk-presence field that is true only when the `.uasset` is
actually on disk, and a `pendingFlush:true` signal when it is dirty-only,
instead of an unqualified `existsAfter:true`. The shared no-op
`McpSafeAssetSave` (and its ~220 corruption-sensitive callers) stays
untouched.

## Evidence (this task — create_metasound_patch "GainStage_Patch")

Friction note verbatim: "create_metasound_patch's save:true did not
actually write the .uasset until a separate save_all (which did persist
the .uasset to disk)." The call log shows the patch was created, then
later `editor.save_all` reports "saved 1" — i.e. the create's own
`save:true` left the package dirty and only the explicit `save_all`
flushed it to disk. The `existsAfter:true` in the create response gave no
hint the asset was memory-only.

(Distinct from `B-metasound-patch-mutators-reject`, the judge-filed bug on
the same task: that is the mutators rejecting a Patch with a false
`ASSET_NOT_FOUND`; this is the *create* path's `save:true` not persisting
to disk — an orthogonal save-fidelity defect on the same handler family.)

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS/save-fidelity friction surfaced by the `audio.authoring.create_metasound_patch` "GainStage_Patch" fuzz task (outcome tool_bug; this save defect is orthogonal to the judge-filed mutator-cast bug). `create_metasound_patch`/`create_metasound_preset` default `save:true` to `McpSafeAssetSave(Patch)` (`MetaSoundPatchPresetHandler.cpp:90-93` / `:217-220`), which only `MarkPackageDirty()` + `AssetCreated()` and never writes the `.uasset` — but the response sets `existsAfter:true` (registry, not disk), implying persistence. The user observed the patch was not on disk until a separate `editor.save_all` ("saved 1") flushed it; the create's own `save:true` gave no hint the asset was memory-only. Same root cause/code as `B-niagara-save-no-disk-write`, which explicitly names the MetaSound path as affected but scopes its fix to niagara only — no MetaSound save ticket exists, so this fills the gap. Proposes the accepted sibling pattern: route MetaSound create `save:true` through the real-save helper `SaveLoadedAssetThrottled`, probe disk presence, and gate via `ShouldTreatAssetSaveAsSuccess` with a `pendingFlush:true` signal when dirty-only — leaving the shared corruption-sensitive `McpSafeAssetSave` untouched.
- `#2-route-create-save-through-disk-write` `IN-REVIEW` developer — Fixed both MetaSound create handlers in `MetaSoundPatchPresetHandler.cpp`: `create_metasound_patch` (was `:90-93`) and `create_metasound_preset` (was `:217-220`) now route `save:true` through the in-tree real-save helper `SaveAssetToDiskReportingPresence(Asset, /*bForce=*/true)` (which wraps `SaveLoadedAssetThrottled` -> `UEditorAssetLibrary::SaveLoadedAsset`, probes `IFileManager::FileSize` for on-disk presence, and gates via `ShouldTreatAssetSaveAsSuccess`) instead of the mark-dirty-only `McpSafeAssetSave`. Each response now carries honest `saveRequested`/`saved` booleans plus `pendingFlush:true` when the asset is dirty-only (mirrors the accepted `niagara.create_*` and `create_level` fixes). The shared corruption-sensitive `McpSafeAssetSave` and its ~220 other callers are untouched. Considered the adversarial suggestion to scope down to a `pendingFlush`-only honesty signal (relying on the existing generic `asset.save`), but a signal-only change leaves `save:true` silently not persisting — strictly worse than the sibling-accepted real-save, and the per-handler real-save reuses the centralized turnkey helper rather than reinventing per-site, so kept the real-save scope. Regression test: added `FMetaSoundCreateSaveWritesToDiskTest` (`PinWright.Assets.MetaSoundCreateSaveWritesToDisk`) to `Tests/Assets/TestMetaSoundPatchPreset.cpp` — factory-creates a real MetaSound patch, asserts no `.uasset` on disk pre-save, then drives the production helper `SaveAssetToDiskReportingPresence` and asserts the `.uasset` genuinely lands on disk (`IFileManager::FileSize >= 0`, reported `OutSize > 0`); reverting either handler to `McpSafeAssetSave` (mark-dirty only, no file written) fails the disk-presence assertions.
