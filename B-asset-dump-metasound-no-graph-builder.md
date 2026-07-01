---
id: B-asset-dump-metasound-no-graph-builder
title: "asset.dump MetaSoundSource misclassified as SoundWave; MetaSoundPatch document is a 41KB single-line ExportText blob"
status: DONE
severity: High
category: bug
tags: [asset-dump, audio, metasound]
---

# asset.dump MetaSoundSource misclassified as SoundWave; MetaSoundPatch document is a 41KB single-line ExportText blob

Two related issues:

(1) `UMetaSoundSource` extends `USoundWaveProcedural : USoundWave`, so the SoundWave dispatch path matches it and writes a useless `sound_wave.json` with sentinel values (`duration: 9999, sampleRate: 0`).

(2) The actual MetaSound graph (~24 MetaSoundPatch + ~112 MetaSoundSource assets) lives in `properties.json` as a single 41KB-scale escaped-text blob (`RootMetasoundDocument`), which is unparseable for an LLM consumer.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Audio/MetaSounds/MS_PlayOneShot_2ch/sound_wave.json` — sentinel `duration: 9999`, `sampleRate: 0`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Audio/MetaSounds/lib_DovetailClip/properties.json` — line 48 is 41,820 chars on a single line (RootMetasoundDocument as escaped ExportText).

**Fix (proposed):** In `AssetDumpHandler.cpp` (line ~626), `Cast<USoundWave>(Asset)` succeeds for MetaSoundSource via inheritance. Exclude MetaSoundSource from SoundWave dispatch, and add a dedicated `MetaSoundDumpBuilder` that walks the `FMetasoundFrontend` document API to emit structured node/edge JSON.

## History
- `#1-initial-repro` `OPEN` reporter — MetaSoundSource is misclassified by `Cast<USoundWave>` (inheritance), producing a useless `sound_wave.json` with `duration: 9999, sampleRate: 0`. The real graph survives only as a 41KB escaped single-line `RootMetasoundDocument` string in `properties.json`. Sample paths: `Game/Audio/MetaSounds/MS_PlayOneShot_2ch/sound_wave.json`, `Game/Audio/MetaSounds/lib_DovetailClip/properties.json` (line 48: 41,820 chars). ~24 MetaSoundPatch + ~112 MetaSoundSource assets affected.
- `#2-add-metasound-dump-builder` `IN-REVIEW` developer — added `MetaSoundDumpBuilder.{h,cpp}` reading `IMetaSoundDocumentInterface::GetConstDocument()`; new upper-chain dispatch branch in `AssetDumpHandler.cpp` (above SoundWave) handles `UMetaSoundPatch`/`UMetaSoundSource`, drops the 41KB `RootMetasoundDocument` blob from `properties.json`, and emits structured `metasound.json` with nodes/edges/variables/rootGraph; regression test `TestMetaSoundDumpBuilder.cpp` covers builder shape and end-to-end dispatch including the `sound_wave.json`-absence guard.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/Game/Audio/MetaSounds/MS_PlayOneShot_2ch` (UMetaSoundSource) and `/Game/Audio/MetaSounds/lib_DovetailClip` (UMetaSoundPatch) each emit `{meta,properties,metasound}.json` only — no `sound_wave.json`. `lib_DovetailClip/properties.json` is 39 lines, max line length 63 chars (was 41,820 chars on line 48). `metasound.json` is structured with `assetKind`, `assetPath`, `edges` (fromNodeID/fromVertexID/toNodeID/toVertexID), nodes/variables/rootGraph.
