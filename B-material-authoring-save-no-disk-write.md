---
id: B-material-authoring-save-no-disk-write
title: "material.authoring create_material / compile_material / create_material_instance (save:true) report saved/existsAfter:true but SaveMaterialAsset only marks dirty — nothing reaches disk, so the materials vanish on cold restart"
status: IN-REVIEW
severity: Critical
category: bug
tags: [material, material-authoring, create-material, compile-material, material-instance, add-landscape-layer, landscape-layer-info, save, save-material-asset, no-disk-write, silent-failure, false-success, cold-load, persistence]
encounters: 2
lastSeen: 2026-07-05T01:25:36.0583337+03:00
---

# material.authoring create/compile/instance claim success but write nothing to disk — materials are lost on restart

The `material.authoring` save path is a **mark-dirty no-op**. `create_material`,
`compile_material`, and `create_material_instance` all take a `save` param
defaulting to `true`, then return a success payload that implies the `.uasset` is
on disk — `create_material` an `AddAssetVerification` block (`existsAfter:true`),
`compile_material` an explicit `saved:true`. It is not. With `save:true` the
handlers only **mark the package dirty + register it with the asset registry**;
no package-save API is ever called. The asset exists only in memory + the registry
(so the in-session `get_material_info` / `get_material_instance_info` readbacks
work), but after the editor closes (or a fuzz `git reset --hard`) **the asset is
gone**, with no error ever surfaced.

This is **cold-load-confirmed asset loss**: a build-glowing-sci-fi-wall-panels
task created a master material `/Game/SciFi/Materials/M_EnergyPanel` (Unlit;
GlowColor + GlowIntensity + PanSpeed params; Panner→ComponentMask→Frac scroll
into EmissiveColor) and a `red alert` child instance
`/Game/SciFi/Materials/MI_EnergyPanel_RedAlert` (GlowColor→{1,0,0,1},
GlowIntensity→12). `create_material` succeeded, `compile_material {save:true}`
returned `compiled:true, compiledWithErrors:false, saved:true`, and
`create_material_instance {save:true}` succeeded; every in-session verify
(`get_material_info`, `get_material_instance_info`) confirmed the params, the
EmissiveColor wiring, and the instance's parent + both overrides. On a real cold
editor restart (no baseline restore) **both** `editor.open_asset` calls failed
with `[ASSET_NOT_FOUND]`, and `Content/SciFi/Materials/` is entirely absent on
disk — neither package was ever persisted. The whole "reusable master + alert
instance" workflow is a silent no-op as far as durable state is concerned, even
though every call reported success.

## Root cause (verified in source)

`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp`

`SaveMaterialAsset` (`:271-277`) is a **mark-dirty no-op** — despite the name it
never calls any package-save API:

```cpp
static bool SaveMaterialAsset(UMaterial* Material)
{
    if (!Material) return false;
    // UE 5.7: Do NOT call SaveAsset - triggers modal dialogs that crash D3D12RHI.
    Material->MarkPackageDirty();
    return true;
}
```

`SaveMaterialFunctionAsset` (`:279-284`) and `SaveMaterialInstanceAsset`
(`:286-291`) are the same `MarkPackageDirty()`-only body. There is no
`UPackage::Save` / `UEditorAssetLibrary::SaveLoadedAsset` / `SavePackage`
anywhere in the helpers. Every material create/compile `save:true` routes through
them and then reports success from the registry, not the disk:

- `create_material` — `:565` `if (bSave) SaveMaterialAsset(NewMaterial);`, then
  `FAssetRegistryModule::AssetCreated` (`:562`) + `AddAssetVerification` (`:568`)
  reports `existsAfter:true` off the registry.
- `compile_material` — `:687` (`:2687`) `if (bSave) SaveMaterialAsset(Material);`,
  then **unconditionally** `Result->SetBoolField(TEXT("saved"), bSave);` (`:2698`)
  — `saved` echoes the *requested* flag, never disk presence.
- `create_material_instance` — `:1820` `if (Ctx.GetBool(TEXT("save"), true)) SaveMaterialInstanceAsset(NewInstance);`,
  then `AddAssetVerification` (`:1824`).
- plus the other `SaveMaterialAsset` callers — `create_material_function`,
  `duplicate_material`/clone paths (`:2403`, `:2442`, `:2481`, `:2640`) and the
  per-edit save sites (`:597`, `:629`, `:661`).

So the asset is registered (`existsAfter:true` / `saved:true`, `get_material_info`
reads it) but never lands on disk. The `save:true` default + the success payload
together imply a persistence that did not happen, and the response carries no
`pendingFlush` / disk-presence signal to say otherwise. `editor.save_all` is the
only thing that actually flushes these dirty packages, and nothing tells the agent
that is required.

This is the **material-authoring analog** of `B-audio-create-save-no-disk-write`
(`SaveAudioAsset`), `B-niagara-save-no-disk-write` / `B-metasound-create-save-no-disk-write`
(`McpSafeAssetSave`), and `B-create-level-saved-true-no-umap` — but on a **distinct
fourth no-op helper trio**: `SaveMaterialAsset` / `SaveMaterialFunctionAsset` /
`SaveMaterialInstanceAsset`. Each sibling ticket scopes its fix to its own
namespace and leaves the material create/compile path unfixed — there is no
material save-no-disk-write ticket, so this fills the gap.

## Additional affected site — `add_landscape_layer` (still un-fixed after #2)

`material.authoring.add_landscape_layer` (`:2509`) creates a
`ULandscapeLayerInfoObject` (one per weight-blended landscape layer) and persists
via its OWN inline path — NOT the `SaveMaterial*` trio the `#2-fix` rerouted — so
the fix missed it entirely. At HEAD the save:true branch is still a mark-dirty
no-op:

```cpp
bool bSave = Ctx.GetBool(TEXT("save"), true);   // :2576
if (bSave)
{
    LayerInfo->MarkPackageDirty();              // :2579  — bare, no real save
}
FAssetRegistryModule::AssetCreated(LayerInfo);  // :2582
...
AddAssetVerification(Result, LayerInfo);        // :2585  — existsAfter:true off the registry
```

It never calls `SaveMaterialAssetToDisk` / `SaveAssetToDiskReportingPresence`, so a
`save:true` (the default) layer-info asset lives only in memory + the asset
registry and is lost on cold restart — the exact defect the material
create/compile path had before `#2`, on a distinct fourth save site. Fix: route
this through `SaveMaterialAssetToDisk` + honest `AddAssetSaveReport` exactly like
`create_material` (`:574-577`), instead of the bare `MarkPackageDirty()` +
unqualified `existsAfter:true`. Cold-load confirmed below (three lost layer-info
assets). The `#2-fix`'s "layer/blend" coverage claim does NOT include this method.

## Cold-load repro (confirmed by a real editor restart)

A `CorruptionCheck` cold-restart (fresh detached headless editor launched after
the attempt's editor dropped; no baseline restore) reopened the two saved assets:

```
editor.open_asset /Game/SciFi/Materials/M_EnergyPanel           → open_ok:false, [ASSET_NOT_FOUND] Asset not found
editor.open_asset /Game/SciFi/Materials/MI_EnergyPanel_RedAlert → open_ok:false, [ASSET_NOT_FOUND] Asset not found
```

The editor stayed fully responsive (each `open_asset` returned a clean structured
`ASSET_NOT_FOUND`; the namespace index returned afterward) — not an editor crash.
Disk confirms the cause: `Content/SciFi/Materials/` is absent on disk (a recursive
`find Content -iname *EnergyPanel*` returns nothing), so neither package was ever
persisted by the create/compile calls. Both are Materials (non-Blueprint, so the
compile readback is N/A). Net: the assets the calls reported saving (`saved:true`
/ `existsAfter:true`) did not survive cold load because they were never written to
disk — they vanish entirely on restart.

severity rationale: impact=corruption × reach=every-session -> Critical

## What it should do

A `save:true` create/compile must persist to disk for real (or, if deferred, the
response must say so — never an unqualified `saved:true` / `existsAfter:true`).
Mirror the accepted sibling fixes (`B-audio-create-save-no-disk-write` #2,
`B-niagara-save-no-disk-write` #2, `B-create-level-saved-true-no-umap`):

- Route the material create/compile `save:true` path through the in-tree real-save
  helper (`SaveAssetToDiskReportingPresence` / `SaveLoadedAssetThrottled` in
  `Utils/AssetUtils.cpp`, which calls `UEditorAssetLibrary::SaveLoadedAsset` behind
  the integrity gate) instead of the mark-dirty `SaveMaterialAsset` trio. Materials
  / MaterialInstances / MaterialFunctions are non-Blueprint/non-SCS, so the
  bulkdata-corruption vector that forced `McpSafeAssetSave` on Blueprint edits
  (`B-bp-saved-state-corruption-mcp-edits`) does not apply. (The existing
  `// Do NOT call SaveAsset - triggers modal dialogs that crash D3D12RHI` comment is
  about the interactive `SaveAsset` modal — the headless `SaveLoadedAsset`/`SavePackage`
  path the siblings use does not raise that dialog.)
- After the save, probe on-disk presence (`IFileManager::FileSize(PackageFilename)`)
  and gate the result through the shared `ShouldTreatAssetSaveAsSuccess` predicate:
  report a `saved` field that is `true` only when the `.uasset` is actually on disk,
  and a `pendingFlush:true` signal when it is dirty-only — instead of `compile_material`
  echoing the requested flag and `create_material` reporting an unqualified
  `existsAfter:true`.

## Workaround

After the create/compile calls, run `editor.save_all` to flush the dirty material
packages to disk before the editor closes or any `git reset --hard`. Nothing in the
create/compile responses signals this is required.

## History
- `#3-additional-cold-load` `IN-REVIEW` reporter — Additional evidence (SAME no-disk-write family, DISTINCT un-fixed save site): the `#2-fix` rerouted the three `SaveMaterial*` helpers but did NOT touch `material.authoring.add_landscape_layer`, whose `ULandscapeLayerInfoObject` creation still ends in a bare inline `LayerInfo->MarkPackageDirty()` (`MaterialAuthoringHandler.cpp:2576-2582`, the dirty at :2579) with no `SaveMaterialAssetToDisk` / `SaveAssetToDiskReportingPresence` call — it then reports `existsAfter:true` from `FAssetRegistryModule::AssetCreated(LayerInfo)` (:2582) + `AddAssetVerification` (:2585), a registry/memory fact, not disk presence. A REALISM-mode landscape-master-material task created three layer-info assets — Grass (h=0.3), Rock (h=0.85), Sand (h=0.5) — at `/Game/Landscape/Layers` via `add_landscape_layer` (save default true), wired them as weight-blended slots via `configure_layer_blend`, compiled `M_Terrain_Master` clean, and `asset.validate`'d all three (all warm-session green). A real cold editor restart (no baseline restore) then found ALL THREE entirely absent: no `.uasset` on disk under `Content/Landscape/Layers`, `/Game/Landscape/Layers` unregistered in the cold asset registry (`asset.list` pathExistsInRegistry:false, 0 assets), and `asset.exists` / `asset.validate` / `asset.get` / `open_asset` all return `[ASSET_NOT_FOUND]`. Control: the sibling Material `M_Terrain_Master` (force-saved via `asset.save`) persisted and cold-opened fine, and its cold MGIR readback matched warm (3 ScalarParameter weight nodes Rock=0.0/Grass=1.0/Sand=0.0) — proving the cold registry loaded fully and the missing layers are a genuine loss, not an incomplete scan. So the `#2-fix`'s "layer/blend" coverage claim is INCOMPLETE for `add_landscape_layer`; route its save:true path through `SaveMaterialAssetToDisk` and report `saved`/`pendingFlush` via `AddAssetSaveReport` like the create_material fix. Editor stayed healthy throughout (cold outcome load_failed, not a crash).
- `#2-fix` `IN-REVIEW` developer — Root-cause fix: the three file-local save helpers in `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp` — `SaveMaterialAsset` (:271), `SaveMaterialFunctionAsset` (:279), `SaveMaterialInstanceAsset` (:286) — were `MarkPackageDirty()`-only no-ops; all three now route through the in-tree real-save helper `SaveAssetToDiskReportingPresence` (forced `SaveLoadedAsset` + `IFileManager::FileSize` disk probe, gated by `ShouldTreatAssetSaveAsSuccess`), the same path the audio/niagara/metasound create fixes adopted. This persists every material `save:true` site (create/compile/instance, plus set-prop, function, layer/blend, duplicate/clone) to disk for real; the stale `// Do NOT call SaveAsset - triggers modal dialogs that crash D3D12RHI` warning was about the interactive modal, not the headless path. Honest reporting wired into the three flagged responses via `AddAssetSaveReport(Result, bSave, bSavedToDisk)` (`saveRequested`/`saved`-gated-on-disk-presence/`pendingFlush`): `create_material` (was unqualified `existsAfter:true` only), `create_material_instance` (now registers via `FAssetRegistryModule::AssetCreated` BEFORE saving, mirroring the audio order), and `compile_material` (replaced the unconditional `SetBoolField("saved", bSave)` echo). Files: `MaterialAuthoringHandler.cpp`. Regression test: `Source/PinWright/Private/Tests/Material/TestMaterialCreateSaveWritesToDisk.cpp` (`PinWright.Material.CreateMaterial.SaveWritesToDisk` / `.NoSaveLeavesNoDiskFile`) drives the real registered `material.authoring.create_material` handler through the dispatcher and FileSize-probes the `.uasset`: save:true must land on disk + report `saved:true`; save:false must write nothing + report `saved:false`. Fails if the helper is reverted to the mark-dirty no-op. Mirrors the accepted sibling test `TestAudioCreateSaveWritesToDisk.cpp`.
- `#1-initial-repro` `OPEN` reporter — COLD-LOAD-confirmed asset loss from a REALISM-mode glowing-sci-fi-panels task (master `/Game/SciFi/Materials/M_EnergyPanel` Unlit with GlowColor/GlowIntensity/PanSpeed params + Panner→ComponentMask→Frac→EmissiveColor scroll; child instance `/Game/SciFi/Materials/MI_EnergyPanel_RedAlert` overriding GlowColor→{1,0,0,1} + GlowIntensity→12). `create_material` succeeded; `compile_material {save:true}` returned `compiled:true, compiledWithErrors:false, saved:true`; `create_material_instance {save:true}` succeeded; in-session `get_material_info` / `get_material_instance_info` confirmed the params, EmissiveColor wiring, parent, and both overrides. A real cold editor restart (fresh headless editor, no baseline restore) then failed `editor.open_asset` on BOTH assets with `[ASSET_NOT_FOUND]`; `Content/SciFi/Materials/` is absent on disk — the packages were never persisted. Root cause: `SaveMaterialAsset` (`MaterialAuthoringHandler.cpp:271-277`) is a mark-dirty no-op (`MarkPackageDirty()` only, explicit `// Do NOT call SaveAsset` comment, no package-save API), and the `SaveMaterialFunctionAsset` (`:279-284`) / `SaveMaterialInstanceAsset` (`:286-291`) variants are identical. `create_material` reports `existsAfter:true` from `FAssetRegistryModule::AssetCreated` (`:562`) + `AddAssetVerification` (`:568`); `compile_material` reports `saved` by echoing the requested `bSave` flag unconditionally (`:2698`) regardless of disk presence; `create_material_instance` saves via `SaveMaterialInstanceAsset` (`:1820`). Same defect shape as `B-audio-create-save-no-disk-write` / `B-niagara-save-no-disk-write` / `B-metasound-create-save-no-disk-write` / `B-create-level-saved-true-no-umap`, but on a distinct fourth no-op helper trio (`SaveMaterialAsset`/`SaveMaterialFunctionAsset`/`SaveMaterialInstanceAsset`) that those siblings leave unfixed. The cold restart IS the replay-confirmation (asset loss the save-time integrity gate let through). Dedup: ripgrep over OPEN + closed found no material save-no-disk-write ticket — the existing material tickets cover shader-compile false success (`B-compile-material-false-shader-success`, DONE), stub handlers (`B-material-stub-handlers-silent-success`), open-editor clobber (`B-material-graph-edit-clobbered-by-open-editor`), and various readback/ergonomic gaps — none touch the disk-write defect.
