---
id: B-asset-dump-non-texture2d-no-native-summary
title: "asset.dump no typed summary for non-2D UTexture assets"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, texture, native-summary]
---

# asset.dump no typed summary for non-2D UTexture assets

`F-asset-dump-native-summary-aspects` shipped `texture_2d.json` with size/pixelFormat/compression/lodGroup/srgb/etc. for `UTexture2D`, but `AssetDumpHandler.cpp` only matches `UTexture2D`. Sibling `UTexture` assets fall through with no typed summary, including TextureCube, TextureCubeArray, Texture2DArray, TextureRenderTarget2D, TextureRenderTargetCube, and VolumeTexture.

`USparseVolumeTexture` is not a `UTexture` subclass in UE 5.6 and is out of scope for this ticket.

Their dimensions/format only survive as embedded ExportText of the `Source` UPROPERTY.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Materials/HDRI/T_HDRI_SandBox/` — TextureCube, only meta.json + properties.json.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Lighting/HDR/circus_maximus_2_4k/` — TextureCube.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/EngineSky/VolumetricClouds/VT_PerlinWorley_Balanced/` — VolumeTexture.
4. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Blueprints/RenderTarget/CRT_Mirror/` — TextureRenderTargetCube.
5. Observe: no `texture.json` typed sidecar.

**Fix (proposed):** Add a generic `texture.json` sidecar for every `UTexture` asset while preserving the existing `texture_2d.json` sidecar for `UTexture2D` compatibility.

## History
- `#1-initial-repro` `OPEN` reporter — `AssetDumpHandler.cpp:622` only matches `UTexture2D`; sibling UTexture subclasses get no typed summary. Sample paths: `App/Materials/HDRI/T_HDRI_SandBox/` (TextureCube), `Game/Lighting/HDR/circus_maximus_2_4k/` (TextureCube), `Game/EngineSky/VolumetricClouds/VT_PerlinWorley_Balanced/` (VolumeTexture), `Game/Blueprints/RenderTarget/CRT_Mirror/` (TextureRenderTargetCube). Dimensions/format only survive in escaped `Source` ExportText.
- `#2-generic-texture-sidecar` `IN-REVIEW` developer — Added generic UTexture `texture.json` sidecar while preserving `texture_2d.json` for UTexture2D; updated dump canonical files, asset wiki, and regression coverage for non-Texture2D assets.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/Game/Lighting/HDR/circus_maximus_2_4k` (TextureCube), `/Game/EngineSky/VolumetricClouds/VT_PerlinWorley_Balanced` (VolumeTexture), and `/Game/Blueprints/RenderTarget/CRT_Mirror` (TextureRenderTargetCube) each emit `texture.json` with `kind`/`textureClass`/`pixelFormat`/`compressionSettings`/`size`/`source`/`srgb`/`lodGroup`/`mipGenSettings`. UTexture2D (`/Engine/EngineMaterials/T_Default_Material_Grid_M`) emits both `texture.json` and `texture_2d.json`, confirming backward compatibility.
