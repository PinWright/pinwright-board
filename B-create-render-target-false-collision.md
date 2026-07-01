---
id: B-create-render-target-false-collision
title: "texture.create_render_target false 'Asset with this name already exists' collision rejects brand-new paths (self-inflicted FindPackage probe)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [texture, render-target, create, false-positive, collision, findpackage]
encounters: 1
lastSeen: 2026-06-30T22:31:27.0074581+03:00
---

# texture.create_render_target — its own collision check trips on a package it just created

`texture.create_render_target` rejects brand-new names/paths — even ones in
folders that do not exist — with `[TEXTURE_ERROR] Asset with this name already
exists: <FullPath>` and never creates the render target. The collision guard
false-positives on a package its own load-probe materialized.

## Workaround (this is why severity is Medium, not a hard blocker)

A render target **can** still be created today via the sibling verb
`render.create_render_target` (RenderHandler.cpp:114), which has no
StaticLoadObject/FindPackage guard — it goes straight to `CreatePackage` +
`NewObject<UTextureRenderTarget2D>` + `FAssetRegistryModule::AssetCreated` and
returns success with `assetPath`. It omits `McpSafeAssetSave`, so the RT is
in-memory + registry-visible only until a follow-up `asset.save` persists it.
So "make a render target" is achievable in-namespace; the defect is that the
**documented `texture.*` creator** is broken and the two near-duplicate creators
diverge. (A future cleanup could dedupe `texture.create_render_target` onto the
`render.*` creator or a shared helper.)

## Root cause (source, ground truth)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp`,
the `create_render_target` branch, collision guard (was lines 2610-2622):

```cpp
// Check for existing asset collision before creating
UObject* ExistingAsset = StaticLoadObject(UTextureRenderTarget2D::StaticClass(), nullptr, *FullPath);   // 2611
if (ExistingAsset)
{
    TEXTURE_ERROR_RESPONSE(FString::Printf(TEXT("Render target already exists: %s"), *FullPath));        // 2614
}

// Also check for any asset with same name (different class collision)
UPackage* ExistingPackage = FindPackage(nullptr, *FullPath);                                             // 2618
if (ExistingPackage)
{
    TEXTURE_ERROR_RESPONSE(FString::Printf(TEXT("Asset with this name already exists: %s"), *FullPath));  // 2621
}
```

Empirically (see repro) the error that fires is the **line-2621** (`FindPackage`)
branch, not the line-2614 (`StaticLoadObject` returned an object) branch:
`ExistingAsset` is null (the asset genuinely does not exist) yet `FindPackage`
returns non-null. The most consistent explanation is that the `StaticLoadObject`
at line 2611, attempting to load a package file that does not exist on disk,
leaves an **orphan empty `UPackage` in memory** as a failed-load side effect; the
`FindPackage(nullptr, *FullPath)` two lines later then finds that very orphan and
aborts. The codebase already documents this exact trap and deliberately avoids it
— `AssetUtils.h`'s `PrepareBlueprintPackageGuardingNameCollision` uses a
registry/load existence check **before** package creation and a
`FindObject(Package, *Name)` **after**, never a pre-CreatePackage
`FindPackage(nullptr, ...)`, precisely because the latter false-positives on a
just-created/orphan package. Control never reaches `CreatePackage` (2625) /
`NewObject` (2632) / `AssetCreated` + `McpSafeAssetSave` (2640-2641), so nothing
is created or saved. (The adversarial review disputes the precise UE internals,
but the repro is deterministic regardless of the exact mechanism: the
load-probe-then-`FindPackage` guard rejects brand-new paths.)

## Fix

Replace the broken `StaticLoadObject` + `FindPackage` double-guard with a
registry-only `UEditorAssetLibrary::DoesAssetExist(FullPath)` existence check
(the plugin's pervasive collision-check idiom — AssetUtils.cpp:146,780,868,1107;
EffectHandler.cpp:181; etc.). It has no load side effect, so it cannot fabricate
the orphan package the old probe tripped on, and it still catches a real prior
asset of any class at the path. A real collision returns an error; a brand-new
path proceeds to create.

## Repro (replayed at HEAD, fresh cold-restarted editor)

All three paths confirmed non-existent (`asset.exists` -> `exists:false`) and the
target folders did not exist:

1. `texture.create_render_target {name:"RT_SecurityCam", path:"/Game/Cameras", width:1024, height:1024}`
   -> `[TEXTURE_ERROR] Asset with this name already exists: /Game/Cameras/RT_SecurityCam`
2. `texture.create_render_target {renderTargetPath:"/Game/OracleProbe/RT_BrandNewProbe_zzq", width:1024, height:1024}`
   -> `[TEXTURE_ERROR] Asset with this name already exists: /Game/OracleProbe/RT_BrandNewProbe_zzq`
3. `texture.create_render_target {renderTargetPath:"/Game/NeverTouched9931/RT_FirstContact", width:256, height:256}`
   (path never probed by anything beforehand — first call to ever name it)
   -> `[TEXTURE_ERROR] Asset with this name already exists: /Game/NeverTouched9931/RT_FirstContact`

`asset.exists {assetPath:"/Game/Cameras/RT_SecurityCam"}` -> `{"exists":false}`
both before and after — confirming the false positive and that no asset is
created.

## Relation to the cold-load ASSET_NOT_FOUND symptom

A cold-restart check found four would-be render targets
(`/Game/Cameras/RT_SecurityCam`, `/Game/Cameras/RT_SecurityCamFeed`,
`/Game/Cameras/RT_SecurityCamMonitor_a17`, `/Game/SecurityCameras/RT_FeedZ9`)
absent from disk after a restart. That is **not** asset corruption / a save-time
integrity-gate miss: it is a direct downstream symptom of this bug — every
`texture.create_render_target` failed up front with the false collision, so
nothing was ever written. No save path is involved; do not chase a phantom
corruption defect.

severity rationale: impact=hard-blocker for the `texture.*` render-target creator
(rejects 100% of valid input) **but** an in-namespace workaround exists
(`render.create_render_target` + `asset.save`), which downgrades impact to a
blocker-with-workaround x reach=normal (render-target creation is a common but
not every-session path) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — `texture.create_render_target` replay-confirmed broken at HEAD on a fresh cold-restarted editor: three never-existed paths (`/Game/Cameras/RT_SecurityCam`, `/Game/OracleProbe/RT_BrandNewProbe_zzq`, `/Game/NeverTouched9931/RT_FirstContact`, all `asset.exists:false`, including one never probed beforehand) each failed with `[TEXTURE_ERROR] Asset with this name already exists: <path>` and created nothing. Source ground truth `TextureHandler.cpp:2610-2622`: the `StaticLoadObject` collision probe (2611) leaves an orphan empty `UPackage` from the failed load, and the `FindPackage(nullptr, *FullPath)` (2618) then trips on it and returns at line 2621 — a self-inflicted false positive that aborts before `CreatePackage`/`NewObject`/`AssetCreated`/save. The cold-load ASSET_NOT_FOUND for the four security-cam render targets is a downstream symptom of this (nothing was ever written), not corruption. No prior board ticket covers this; `B-texture-create-placeholder-fake-success` is the array/cube/volume fake-success family and explicitly assumes `create_render_target` works.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: severity High->Medium and dropped the "no in-namespace workaround / only documented creator is fully non-functional" claim, because the sibling verb `render.create_render_target` (RenderHandler.cpp:114) creates a render target today with no FindPackage guard (+ `asset.save` to persist). Also softened the orphan-`UPackage` mechanism to empirically-grounded (the adversarial lens disputes the precise UE internals; the deterministic repro stands either way). Fixed as proposed: replaced the `StaticLoadObject`(2611)+`FindPackage(nullptr,...)`(2618) double-guard in `Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp` (`create_render_target` branch) with a registry-only `UEditorAssetLibrary::DoesAssetExist(FullPath)` check (no load side effect; still catches a real same-path collision). Added regression test `PinWright.texture.create_render_target.NewPathSucceeds` (`FTextureCreateRenderTargetNewPathSucceedsTest` in `Source/PinWright/Private/Tests/Assets/TestMaterialHandlers.cpp`): creates an RT at a unique never-existing `/Game/PinWrightTests/RenderTargets/RT_NewPath_<guid>`, asserting `DoesAssetExist`==false before, the handler returns `bSuccess`==true (not the false-collision error), and the asset exists after; cleans up the asset. Reverting the fix flips that success assertion to a failure. Not compiled/tested here (later phase does).
</content>
</invoke>
