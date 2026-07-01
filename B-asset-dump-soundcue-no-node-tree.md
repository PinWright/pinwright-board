---
id: B-asset-dump-soundcue-no-node-tree
title: "asset.dump SoundCue dumps lack node-tree structure — only flat AllNodes path-string list"
status: DONE
severity: High
category: bug
tags: [asset-dump, audio, soundcue]
---

# asset.dump SoundCue dumps lack node-tree structure — only flat AllNodes path-string list

SoundCue dump contains only a flat `AllNodes` list of subobject path strings (`":SoundNodeWavePlayer_0"`, etc.) plus a `FirstNode` pointer. No graph edges, no per-node properties, no SoundWave references resolved.

Reconstructing a SoundCue from the dump requires N additional `asset.dump` calls per child node, defeating the cache's purpose for audio review.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Audio/Cue_AssemblyLine/properties.json`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Audio/Cue_Drone/properties.json`.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Audio/Cue_MainAmbient/properties.json`.
4. Observe: `AllNodes` is a list of subobject path strings; no edges, no per-node configuration.

**Fix (proposed):** No `SoundCueDumpBuilder` next to `SoundWaveDumpBuilder` / `LevelSequenceDumpBuilder`. SoundCues fall through to generic `BuildClassPropertyJson` which only flattens UPROPERTY values; the USoundNode UEdGraph subobject tree isn't traversed. Add a SoundCueDumpBuilder that walks `FirstNode` recursively and emits structured JSON with edges + per-node settings.

## History
- `#1-initial-repro` `OPEN` reporter — SoundCue dumps contain only a flat `AllNodes` list of subobject path strings plus a `FirstNode` pointer; no graph edges, no per-node properties, no resolved SoundWave references. Sample paths: `Game/Audio/Cue_AssemblyLine/properties.json`, `Game/Audio/Cue_Drone/properties.json`, `App/Audio/Cue_MainAmbient/properties.json`. Reconstructing the cue requires per-node `asset.dump` follow-ups.
- `#2-add-soundcue-dump-builder` `IN-REVIEW` developer — added `SoundCueDumpBuilder.{h,cpp}` and `Cast<USoundCue>` dispatch branch in `AssetDumpHandler.cpp`; emits `sound_cue.json` with `nodes`/`edges`/per-node CDO-diff properties and resolved `SoundWaveAssetPtr`; regression test `TestSoundCueDumpBuilder.cpp` covers builder shape and end-to-end dispatch.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/Game/Audio/Cue_Drone` and `/Game/Audio/Cue_AssemblyLine` both emit `sound_cue.json` alongside `meta.json`/`properties.json`. Cue_Drone shows single WavePlayer node with bLooping=true and `soundWave: /Game/Audio/S_Drone.S_Drone`. Cue_AssemblyLine shows full graph: 14 nodes including Mixer→WavePlayers/Looping→Delay→WavePlayer chains, with `edges` arrays per node, per-node `properties` (DelayMin/Max, InputVolume arrays, bLooping), and resolved `soundWave` paths on every WavePlayer.
