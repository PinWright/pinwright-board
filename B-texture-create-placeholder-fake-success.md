---
id: B-texture-create-placeholder-fake-success
title: "texture.create_texture_array / create_cube_texture / create_volume_texture return fake success and create no asset"
status: IN-REVIEW
severity: High
category: bug
tags: [texture, stub, silent-failure, fake-success, create]
---

# texture.create_{texture_array,cube_texture,volume_texture} are success-returning no-ops

`texture.create_texture_array` (and its siblings `texture.create_cube_texture`
and `texture.create_volume_texture`) reply with a success-shaped response —
a "placeholder created" message and a fabricated `assetPath` — but never
construct a `UTexture2DArray` / `UTextureCube` / `UVolumeTexture`, never create a
`UPackage`, and never register or save anything. The reported `assetPath` does
not exist after the call, so every documented follow-up (`texture.describe`,
the `set_*` configurators, `asset.exists`) fails. This violates the repo's
"never fake-success" rule and is indistinguishable from a working method to any
caller that doesn't read the handler source.

## Why it matters

A caller doing the obvious round-trip — create the array, `texture.describe` it,
then run `set_compression_settings` / `set_texture_group` /
`set_streaming_priority` / `set_texture_filter` / `set_texture_wrap` /
`set_lod_bias` on it — is dead on arrival at step 2: `describe` returns
`[ASSET_NOT_FOUND]` because the asset was never created. The create call's
`success` + `assetPath` actively mislead the caller into believing the asset is
real, so the failure only surfaces on the next call instead of at the create.

## Root cause (source)

`Source/PinWright/Private/Handlers/Material/TextureHandler.cpp`,
`ExecuteTextureAction`:

- `create_texture_array` (lines 2683-2703): computes `FullPath = Path / Name`,
  then hard-codes
  ```cpp
  Response->SetBoolField(TEXT("success"), true);
  Response->SetStringField(TEXT("message"),
      FString::Printf(TEXT("Texture array '%s' placeholder created (%dx%dx%d)"), *Name, Width, Height, NumSlices));
  Response->SetStringField(TEXT("assetPath"), FullPath);
  Response->SetStringField(TEXT("note"), TEXT("Texture arrays typically created from multiple 2D textures."));
  ```
  No `UTexture2DArray` / `UPackage` / `FAssetRegistryModule::AssetCreated` / save.
- `create_cube_texture` (lines 2640-2659): identical pattern, "Cube texture '%s'
  placeholder created", no asset.
- `create_volume_texture` (lines 2661-2681): identical pattern, "Volume texture
  '%s' placeholder created (%dx%dx%d)", no asset.

Contrast the immediately-preceding `create_render_target` branch (ends at line
2637), which actually does `FAssetRegistryModule::AssetCreated(RenderTarget)` +
`McpSafeAssetSave(RenderTarget)` before returning — so the create family is
half-real.

## Fix options (any of)

1. **Implement them.** Create the real `UTexture2DArray` / `UTextureCube` /
   `UVolumeTexture` in a `UPackage` at `FullPath`, call
   `FAssetRegistryModule::AssetCreated` + `McpSafeAssetSave` like
   `create_render_target` does. (Texture arrays are typically built from source
   slices, but an empty/placeholder asset that actually exists in the registry is
   enough to make the round-trip honest.)
2. **Make them fail loud.** Replace `SendSuccess` with
   `SendError("NOT_IMPLEMENTED", "create_texture_array is a stub; build texture
   arrays from source 2D textures via import_texture …")` until a real
   implementation lands. One-line-per-branch removal of the silent-success
   footgun.

Option 2 is the minimum safe change; option 1 closes the capability gap.

## Repro

1. `texture.create_texture_array {name: "T_TerrainArray", path: "/Game/Textures/Terrain", width: 512, height: 512, numSlices: 4}`
   -> `{"message":"Texture array 'T_TerrainArray' placeholder created (512x512x4)","assetPath":"/Game/Textures/Terrain/T_TerrainArray","note":"Texture arrays typically created from multiple 2D textures."}` (success)
2. `texture.describe {assetPath: "/Game/Textures/Terrain/T_TerrainArray"}`
   -> `[ASSET_NOT_FOUND] Could not load texture: /Game/Textures/Terrain/T_TerrainArray`
3. `asset.exists {assetPath: "/Game/Textures/Terrain/T_TerrainArray"}`
   -> `{"success":true,"exists":false,"assetPath":"/Game/Textures/Terrain/T_TerrainArray"}`

(Symmetric repro for `create_cube_texture` and `create_volume_texture` — same
hard-coded-success branches in the same file.)

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `texture.create_texture_array` against the live editor: returned `success` with `message:"...placeholder created (512x512x4)"` and `assetPath:/Game/Textures/Terrain/T_TerrainArray`, but `texture.describe` -> `[ASSET_NOT_FOUND]` and `asset.exists` -> `exists:false`. Source confirms `TextureHandler.cpp:2683-2703` (`create_texture_array`), `2640-2659` (`create_cube_texture`), `2661-2681` (`create_volume_texture`) all hard-code `success:true` + a fabricated `assetPath` without creating a UTexture/UPackage or registering/saving, unlike the adjacent `create_render_target` branch which does. Fake-success violates the never-fake-success rule and breaks the documented create->describe->configure round-trip. No prior board ticket references these create methods.
- `#2-fix-fail-loud` `IN-REVIEW` developer — Applied Fix option 2 (minimum-safe, matching the board playbook — B-material-stub-handlers-silent-success DONE / B-input-trigger-modifier-stub-silent-success). All three create branches in `Source/EditorAutomationRpcGateway/Private/Handlers/Material/TextureHandler.cpp` now fail loud instead of fabricating success: each replaced its `success:true` + `assetPath`/`note` block with a new `TEXTURE_ERROR_RESPONSE_CODE(TEXT("NOT_IMPLEMENTED"), ...)` (new sibling macro of `TEXTURE_ERROR_RESPONSE` that sets `errorCode` so `RunTextureAction` forwards it through `Ctx.SendError`). `create_cube_texture` (was 2640-2659) -> NOT_IMPLEMENTED, points at `texture.import_texture` for HDR cube maps; `create_volume_texture` (was 2661-2681) -> NOT_IMPLEMENTED (VDB/EXR sequence or 2D slices); `create_texture_array` (was 2683-2703) -> NOT_IMPLEMENTED (assemble from 2D source textures). The unused width/height/depth/numSlices/size/path/FullPath locals were dropped; `name` validation is kept so an empty name still returns the existing "name is required" error. Option 1 (real assets) was rejected for now: a renderable cube/volume/array needs real source slices, so an empty placeholder would still not survive `texture.describe` — the capability gap stays open under the wiki ticket E-texture-create-wiki-advertises-stub. Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Material/TestTextureCreateStubsNotImplemented.cpp` — three `IMPLEMENT_SIMPLE_AUTOMATION_TEST` cases invoke each production handler via `InvokeHandlerWithCapture` (through the shared `TestHandlerReturnsNotImplemented`) with a valid `name` and assert `bSuccess==false` and `ErrorCode=="NOT_IMPLEMENTED"`; restoring the fake-success branches flips `bSuccess` true and fails them. Not compiled/run here (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule. The one citation is in a history row and stays verbatim; it has **no successor**. `Tests/Material/TestTextureCreateStubsNotImplemented.cpp` was deleted in plugin `e0d0fe2c` when all three verbs it covered were culled (cull record `E-rpc-cull-151-record`); they grep to zero at HEAD. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
