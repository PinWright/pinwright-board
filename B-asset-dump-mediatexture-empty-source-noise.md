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
- `#4-repoint-citations-after-property-utils-split` `DONE` reporter — Citation maintenance only; **no behavioural claim changes and the status is untouched**. `Utils/PropertyUtils.cpp` was split into `PropertyExport.cpp` / `PropertyImport.cpp` / `PropertyInspection.cpp` / `PropertyDiff.cpp` (`PropertyUtils.h` survives only as a deprecated umbrella forwarder), so `#2`'s `PropertyUtils.cpp` citations were unresolvable paths, not stale line numbers. `#2` is left verbatim per the append-only rule and maps here, verified at plugin HEAD `ef8a1f1b`: the generic `FStructProperty` fallback `PropertyUtils.cpp:460` → `Utils/PropertyExport.cpp:900-903`, inside `ExportPropertyToJsonValue` (`:605`); the `Source`-property skip → the predicate `IsTextureSourceNoisyProperty` (`PropertyExport.cpp:1179-1185`, the owner-class test at `:1184`), consulted from `ShouldEmitClassDumpProperty` at `:1413`, which `BuildClassPropertyJson` (`:1437`) calls at `:1462` — i.e. the skip is no longer inline in `BuildClassPropertyJson` as `#2` describes, but factored into a named predicate one level out. The regression test `FPropertyUtilsTextureSourceSkipTest` still exists, at `Source/PinWright/Private/Tests/Utility/TestPropertyUtilsTextureSourceSkip.cpp:23-25`, test id `PinWright.utils.property_utils.TextureSourceSkip`. **One correction to `#2`'s wording:** at HEAD the generic struct fallback is a `StructToJsonObject` field walk, not the ExportText string `#2` calls "the default-constructed `FTextureSource` placeholder string" — the site is the same, the emission shape changed under it. **Source-side finding, not fixed here:** the test file's own comments carry the same dead citation twice — `TestPropertyUtilsTextureSourceSkip.cpp:3` ("in PropertyUtils.cpp") and `:12-14` ("PropertyUtils.cpp:460-465") — which will mislead the next reader of the test. That wants a source commit, not a board one; it is the third such in-code stale reference found in this sweep, alongside the two on `B-asset-dump-tmap-struct-key-mangled` `#4` and `B-asset-dump-texture2d-duplicate-sidecars` `#5`.
