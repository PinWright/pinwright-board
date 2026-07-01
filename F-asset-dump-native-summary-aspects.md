---
id: F-asset-dump-native-summary-aspects
title: "asset.dump needs semantic summary aspects for common native assets"
status: DONE
severity: Medium
category: feature
tags: [asset, dump, native-assets, summary]
---

# asset.dump needs semantic summary aspects for common native assets

Common native asset classes currently fall through to `meta.json` plus generic `properties.json`. Some useful data is present, but it is buried in raw reflected fields and is not stable or compact enough for broad dump review.

**Workaround:** Manually inspect `properties.json` or call type-specific live RPCs where available.

**Fix:** Add typed summary files with compact, stable, LLM-facing fields for common asset classes.

## History
- `#1-live-sampling` `OPEN` reporter — StaticMesh, Texture2D, SoundWave, and LevelSequence assets currently dump only `meta.json` plus generic `properties.json`. Some useful values exist but are buried as raw reflected fields (`StaticMesh` `ExtendedBounds`/`StaticMaterials`, `Texture` `ImportedSize`/`CompressionSettings`/`Source`, `Sound` `Duration`/`NumChannels`/`SampleRate`), while `LevelSequence` exposes only object references and no track/section/key summary. Add typed summary files such as `static_mesh.json`, `texture.json`, `sound_wave.json`, and `level_sequence.json` with compact LLM-facing fields and stable diff output.
- `#2-shipped-three-of-four-aspects` `IN-REVIEW` developer — Implemented static_mesh.json, texture_2d.json, sound_wave.json sidecars under Private/Handlers/Asset/ via per-class XxxDumpBuilder files; dispatched from BuildAllFilesForAsset, registered in DumpFileNames + Canonical[]. Regression test TestAssetDumpNativeSummaries.cpp covers all three with engine-asset fixtures. ULevelSequence support split into sibling ticket F-asset-dump-level-sequence-summary because it requires MovieScene track/section walking.
- `#3-verify-fix` `DONE` tester — Verified: asset.dump on /Engine/EngineResources/WhiteSquareTexture, /Engine/EngineMeshes/Sphere, /Engine/VREditor/Sounds/UI/Drop_Item_From_ContentBrowser produced texture_2d.json (compressionSettings, pixelFormat, size, source), static_mesh.json (bounds, lods, materials, trianglesByLod=[528], verticesByLod=[323]), sound_wave.json (duration=1.024s, numChannels=2, sampleRate=48000) respectively, all non-empty and structurally compact as described.
