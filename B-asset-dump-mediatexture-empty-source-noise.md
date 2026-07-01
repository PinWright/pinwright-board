---
id: B-asset-dump-mediatexture-empty-source-noise
title: "asset.dump BinkMediaTexture/MediaTexture properties.json Source field is an all-zero placeholder (consumes space, no info)"
status: DONE
severity: Low
category: bug
tags: [asset-dump, properties, mediatexture]
---

# asset.dump BinkMediaTexture/MediaTexture properties.json Source field is an all-zero placeholder (consumes space, no info)

BinkMediaTexture `Source` field is reported with all-zero GUID and no meaningful texture-source metadata: `"(Id=00000000000000000000000000000000,NumLayers=1,BlockDataOffsets=(0))"`.

Provides no usable info but consumes property real estate.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Movies/Controller_Pitch_Tex/properties.json` (line 49).
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Movies/Drone_Pitch_Tex/properties.json`.
3. Observe: `Source` field is the all-zero placeholder.

**Fix:** Skip the `Source` FProperty (FName=`Source`, owner class=`UTexture`) inside `BuildClassPropertyJson`'s per-property loop. The field carries no actionable signal in properties.json for *any* UTexture subclass: Texture2D already gets a dedicated `Texture2DDumpBuilder` sidecar; non-Texture2D textures default-construct it. One-line suppression — riskless. Matches the existing precedent of `IsCompilerManagedUserWidgetFlag` in the same file.

## History
- `#1-initial-repro` `OPEN` reporter — BinkMediaTexture / MediaTexture `Source` field dumps an all-zero-GUID placeholder `(Id=000...,NumLayers=1,BlockDataOffsets=(0))`. Sample paths: `Game/Movies/Controller_Pitch_Tex/properties.json` (line 49), `Game/Movies/Drone_Pitch_Tex/properties.json`. No usable signal.
- `#2-skip-texture-source-property` `IN-REVIEW` developer — Skip `FProperty` named `Source` on owner class `UTexture::StaticClass()` in `BuildClassPropertyJson` (`PropertyUtils.cpp`). The default-constructed `FTextureSource` placeholder string was emitted by the generic FStructProperty fallback at `PropertyUtils.cpp:460`. Added regression test `FPropertyUtilsTextureSourceSkipTest` constructing a transient `UTexture2D` and asserting `Source` is absent from `BuildClassPropertyJson` output.
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/Game/Movies/Controller_Pitch_Tex` and `/Game/Movies/Drone_Pitch_Tex`; grep for `"Source"` key in both resulting `properties.json` returns no matches (only unrelated `SourceData`/`SourceFilePath`/`SourceFileTimestamp` AssetImportData fields remain). The all-zero `(Id=000...,NumLayers=1,BlockDataOffsets=(0))` placeholder is gone.
