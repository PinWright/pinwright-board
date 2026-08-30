---
id: B-niagara-save-no-disk-write
title: "niagara.save (and niagara.create_system / save:true edits) report saved:true but McpSafeAssetSave never writes the .uasset to disk"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, save, create-system, mcp-safe-asset-save, silent-failure, false-success, sizebytes]
---

# niagara.save claims success but writes nothing to disk

`niagara.save` returns `{saved:true, sizeBytes:0}` for a freshly created
Niagara System, but **no `.uasset` file is ever written to disk**. The asset
exists only in memory + the asset registry, so the "save it to disk" final
step of a build-from-scratch task silently does nothing. The same defect
affects `niagara.create_system` (which calls the same save helper internally
and reports `existsAfter:true`) and every `niagara.*` edit RPC invoked with
`save:true`.

The smoking gun is in the response itself: `niagara.save` computes
`sizeBytes` by probing the on-disk package file
(`IFileManager::FileSize(PackageFilename)`), and returns `0` whenever the file
is absent (`RawSize < 0 ? 0 : RawSize`). So `saved:true` with `sizeBytes:0`
is `niagara.save` reporting, in the same payload, that it "saved" a file that
is not on disk. This is a classic silent success-with-no-effect plus a
self-contradicting result.

## Root cause

`McpSafeAssetSave(UObject*)` in `Utils/AssetUtils.cpp:214-226` does **not save
to disk at all** — despite the name it only marks the package dirty and
notifies the asset registry, then unconditionally returns `true`:

```cpp
bool McpSafeAssetSave(UObject* Asset)
{
    if (!Asset) return false;
    // UE 5.7+ Fix: Do not immediately save newly created assets to disk.
    // Saving immediately causes bulkdata corruption and crashes.
    // Instead, mark the package dirty and notify the asset registry.
    Asset->MarkPackageDirty();
    FAssetRegistryModule::AssetCreated(Asset);
    return true;
}
```

There is no `UPackage::Save` / `FEditorFileUtils::PromptForCheckoutAndSave` /
`SavePackage` anywhere in this function. Every caller that treats its `true`
return as "persisted" is wrong:

- `niagara.save` (`NiagaraCompileHandler.cpp:135`) sets `saved:true` from this
  return, then reads `FileSize` of a file that was never written → `0`.
- `niagara.create_system` (`NiagaraHandler.cpp:276`) calls it after
  `CreatePackage` + `NewObject` + `FAssetRegistryModule::AssetCreated`, so the
  system is registered (hence `existsAfter:true`, `niagara.inspect` works) but
  never lands on disk.
- `niagara.add_parameter` / `add_emitter` / `remove_emitter` with `save:true`
  (`NiagaraHandler.cpp:99,214`), plus the Blueprint-create path
  (`AssetUtils.cpp:928`), MetaSound, StateTree, etc. — all report a save that
  did not happen. (These ~220 shared callers are why the fix is scoped to the
  niagara path, not the shared helper — see **Fix scope** below.)

(The Niagara handlers live under `Private/Handlers/Niagara/`, not
`Private/Handlers/AI/`; line numbers above are correct.)

Confirmed in the editor log: an actual save always emits
`LogSavePackage: Moving output files for package: ...` and a `.tmp → Content/
...uasset` move (e.g. `BP_NetPlayer`, the maps, `SM_CrystalShardReplay`
autosave). There is **no** such line for either `FX_Projectile` or
`FX_OracleReplay` — only the `LogNiagara: Compiling System` lines. The
packages are dirtied in memory and never flushed.

This is the asset analog of `B-create-level-saved-true-no-umap` (levels), but
a distinct code path and a worse one: the level path at least calls
`FEditorFileUtils::SaveLevel` and is masked by an OR-policy
(`ShouldTreatLevelSaveAsSuccess`); `McpSafeAssetSave` never attempts a disk
write at all.

The DONE verification of `F-niagara-compile-save-explicit` (#4) that reported
`sizeBytes:702088` was a false positive: it ran `niagara.save` on a
pre-existing on-disk asset (`/Water/Effects/Niagara/Shoreline/NiagaraShore_System`),
so `FileSize` returned the size of the *old* file already on disk — the "save"
still wrote nothing new.

## Why it matters

"Build a Niagara System from scratch and save it to disk" is a basic,
extremely common task — it is the explicit final step of the attempted task.
Because `niagara.save` answers `saved:true`, an agent reasonably believes the
asset is persisted and ends the workflow. After the editor closes (or a fuzz
`git reset --hard`), the asset is gone — all the create/add/rename/set work is
lost with no error ever surfaced. `editor.save_all` is the only way to
actually flush these dirty packages, and nothing tells the agent that is
required.

## Repro (replayed via mcp__editor-automation__call)

1. `niagara.create_system {name:"FX_OracleReplay", savePath:"/Game/Pickups/VFX"}`
   → `{success:true, systemPath:".../FX_OracleReplay.FX_OracleReplay", existsAfter:true, assetClass:"NiagaraSystem"}`.
2. `niagara.add_parameter` User.Speed/User.LifeTime (float), `niagara.set_parameter`
   User.Speed=1200, `niagara.rename_parameter` User.Speed→User.Velocity (compile:true)
   — all succeed; `niagara.inspect` confirms User.Velocity carries 1200 (the
   rename itself is correct; not the bug here).
3. `niagara.save {assetPath:"/Game/Pickups/VFX/FX_OracleReplay"}`
   → `{saved:true, package:"/Game/Pickups/VFX/FX_OracleReplay", sizeBytes:0}`.
4. `niagara.save {assetPath:"/Game/Pickups/VFX/FX_OracleReplay", force:true}`
   → still `{saved:true, ..., sizeBytes:0}`.
5. On-disk check: `Content/Pickups/VFX/` is empty — no `FX_OracleReplay.uasset`
   anywhere under the project; no `LogSavePackage` line for it in
   `Saved/Logs/EAContentExamples57.log` (only the `LogNiagara: Compiling System`
   line). Same for the attempt's `FX_Projectile` (also absent on disk).

## Fix scope (corrected)

**Do NOT change the shared `McpSafeAssetSave` no-op helper.** Its deferred
mark-dirty behaviour is *deliberate* and corruption-driven: the inline comment
("Saving immediately causes bulkdata corruption and crashes",
`AssetUtils.cpp:219-221`) traces to `B-bp-saved-state-corruption-mcp-edits`
(DONE, Critical) — saving Blueprint/widget assets right after MCP edits produced
un-loadable crashing `.uasset` files, and `PinWright_SCSHandlers.cpp`
deliberately switched *from* `SaveLoadedAssetThrottled` *to* `McpSafeAssetSave`
for exactly that reason. The helper is shared by ~220 call sites across 53 files
(Blueprint create, SCS, MetaSound, StateTree, SoundWave, AnimBP, GAS, textures,
…), all of which treat its `true` as "persisted". Mutating it to a real save —
or renaming it — re-exposes every one of those callers to the corruption vector
and is a large, unrelated blast radius. The defect is not the shared helper; it
is that the niagara save path applies a mark-dirty's `true` as an on-disk
`saved:true` for a create-from-scratch flow where the `.uasset` landing on disk
is the only honest persistence signal.

The fix belongs in the niagara save path, mirroring the already-accepted sibling
`B-create-level-saved-true-no-umap` fix (handler-level disk probe + honest
report, shared predicate left intact):

1. **Save for real where it's needed.** `niagara.save` should route through the
   existing in-tree real-save helper `SaveLoadedAssetThrottled`
   (`AssetUtils.cpp:452-518`, which calls `UEditorAssetLibrary::SaveLoadedAsset`
   behind a Blueprint integrity gate) instead of the mark-dirty `McpSafeAssetSave`.
   `niagara.create_system` / `niagara.create_emitter` likewise persist via the
   real-save helper after `AssetCreated`.
2. **Gate `saved` on disk presence, fail loud otherwise.** After the save, probe
   the on-disk package via `IFileManager::FileSize(PackageFilename)` and run the
   result through a new pure predicate
   `ShouldTreatAssetSaveAsSuccess(bSaveReportedSuccess, bFileExistsOnDisk)`
   (added beside `ShouldTreatCreateLevelSaveAsSuccess`). `saved` is `true` only
   when the save reported success AND the `.uasset` is actually on disk. When the
   file is absent, report `saved:false` plus a `pendingFlush:true` signal so the
   agent knows the asset is dirty-only and an `editor.save_all` is required —
   never a `saved:true` with a self-contradicting `sizeBytes:0`.

This makes disk presence load-bearing exactly on the niagara save path while
leaving the corruption-driven shared no-op helper (and its Blueprint/SCS/MetaSound/
StateTree/GAS callers) untouched.

## Cross-ref

- `B-create-level-saved-true-no-umap` (OPEN) — sibling false-`saved:true` for
  levels; different code path (`McpSafeLevelSave` OR-policy vs.
  `McpSafeAssetSave` no-op).
- `F-niagara-compile-save-explicit` (DONE) — shipped `niagara.save`; its #4
  verification's `sizeBytes:702088` was a stale pre-existing on-disk file, not
  proof of a write.
- `F-asset-save` (OPEN) — proposes generic `asset.save`; would share this
  helper and inherit the bug unless fixed.

## History
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD then fix. Board-historian lens (verified) showed the headline Fix option 1 (rewrite the shared `McpSafeAssetSave` to actually save) collides with the corruption history (`B-bp-saved-state-corruption-mcp-edits`, DONE/Critical — immediate save after MCP edits produced un-loadable crashing `.uasset` files; SCS deliberately switched *to* `McpSafeAssetSave` for this) and a ~220-call-site / 53-file blast radius (Blueprint/SCS/MetaSound/StateTree/GAS/textures…). Renaming the helper (option 3) shares that blast radius. So I rewrote the **Fix scope** section to leave the shared no-op helper untouched and retarget the fix to the niagara save path (mirroring the accepted sibling `B-create-level-saved-true-no-umap` #2 handler-probe approach). Also corrected the directory citation (`Private/Handlers/Niagara/`, not `…/AI/`) and the Blueprint-create line (`AssetUtils.cpp:928`). Implementation: (1) added a pure predicate `ShouldTreatAssetSaveAsSuccess(bSaveReportedSuccess, bFileExistsOnDisk)` in `Utils/AssetUtils.cpp`/`.h` (beside `ShouldTreatCreateLevelSaveAsSuccess`) — `saved` requires BOTH a reported success AND the `.uasset` actually on disk; (2) `niagara.save` (`NiagaraCompileHandler.cpp`) now routes through the real-save helper `SaveLoadedAssetThrottled` (`UEditorAssetLibrary::SaveLoadedAsset` behind the Blueprint integrity gate), probes `IFileManager::FileSize`, and reports `saved` via the new predicate, adding `pendingFlush:true` when the file is absent instead of a false `saved:true`+`sizeBytes:0`; (3) `niagara.create_system`/`create_emitter`/`add_emitter`/`remove_emitter` and the shared `FinalizeNiagaraEdit` save path (`NiagaraHandler.cpp` via a file-local `SaveNiagaraAssetReportingDiskPresence` helper; `NiagaraEditTypes.cpp:1582` inline) likewise persist for real and gate `saved`/`pendingFlush` on disk presence. The shared `McpSafeAssetSave` and its ~220 callers are untouched. Files: `Source/EditorAutomationRpcGateway/Private/Utils/AssetUtils.cpp`, `Source/EditorAutomationRpcGateway/Private/Utils/AssetUtils.h`, `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraCompileHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraEditTypes.cpp`. Test: added `FAssetSavePolicyRequiresDiskFileTest` in `Source/EditorAutomationRpcGateway/Private/Tests/Core/TestLevelSaveLoadUtils.cpp` exercising the production `ShouldTreatAssetSaveAsSuccess` — asserts `(true,false)`→false (reported-success-but-no-disk-file, the defect), `(true,true)`→true, and false for both `(false,*)`; reverting a handler to `saved = McpSafeAssetSave(Asset)` (always true regardless of disk) regresses this `(true,false)`→false contract. Also corrected the stale comment in `Tests/Niagara/TestNiagaraCompileSave.cpp` that claimed `McpSafeAssetSave` returns false on a transient package. Did not compile/run (later phase).
- `#1-initial-repro` `OPEN` reporter — Replayed the full create→edit→rename→save chain on `/Game/Pickups/VFX/FX_OracleReplay`. `niagara.save` returned `{saved:true, sizeBytes:0}` for both `force:false` and `force:true`; no `.uasset` was written anywhere under the project and no `LogSavePackage` line appears for it (only `LogNiagara: Compiling System`). Same for the attempt's `FX_Projectile`. Root cause: `McpSafeAssetSave` (`Utils/AssetUtils.cpp:214-226`) never calls any package-save API — it only `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` and returns `true` unconditionally; `niagara.save` (`NiagaraCompileHandler.cpp:135`) and `niagara.create_system` (`NiagaraHandler.cpp:276`) both rely on that return. `sizeBytes:0` is `IFileManager::FileSize` returning -1 for the absent file, clamped to 0. The seed `niagara.rename_parameter` itself is correct (value 1200 preserved, references updated, collision-checked) — the culprit is `niagara.save`/`McpSafeAssetSave`. Dedup: distinct from `B-create-level-saved-true-no-umap` (level path, `McpSafeLevelSave` OR-policy) and from `F-niagara-compile-save-explicit` DONE (whose `sizeBytes:702088` was a stale on-disk file). Ripgrep + qmd found no existing Niagara save-to-disk ticket.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 6 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. One file basename renamed by the same plugin commit is repointed with it (`EditorAutomationRpcGateway.Build.cs` → `PinWright.Build.cs`, `EditorAutomationRpcGateway_SCSHandlers` / `_BlueprintHandlers_List` → `PinWright_*`), verified present at HEAD. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
