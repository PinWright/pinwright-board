---
id: B-scir-missing-audio-and-attenuation
title: "SoundCue IR missing audio metadata (sample rate, channels, attenuation, branch nodes)"
status: DONE
severity: Medium
category: bug
tags: [scir, soundcue, decompiler]
---

# SoundCue IR missing audio metadata

Related to but distinct from [B-asset-dump-soundcue-no-node-tree](B-asset-dump-soundcue-no-node-tree.md). Where that ticket covers missing node tree structure, this one covers per-node and per-wave audio metadata:

- Sample rate, channel count, duration on `SoundWavePlayer` nodes
- `Attenuation` settings on attenuation nodes (distance falloff curve, spatialization)
- `Branch` condition node properties
- `Modulator` node properties (envelope shape, range)

Currently SCIR only captures wave-player asset references and loop flags.

## Sample

`App/Audio/Cue_Drone/scir.txt` — volume/pitch present, but no audio-spec data, no attenuation, no branch.

## Fix sketch

`SCIRDecompiler.cpp` (or equivalent) — extend the node-emission path to walk each SoundCue node type and emit its properties. Mirror what the existing JSON-side dumpers do, in DSL form.

## History
- `#1-scir-audio-metadata-gap` `OPEN` reporter — companion to B-asset-dump-soundcue-no-node-tree; covers per-node audio metadata that's needed for understanding SoundCue behavior.
- `#2-audio-metadata-scir` `IN-REVIEW` developer — Added targeted SCIR emission for wave audio metadata, attenuation settings, branch parameters, and modulator ranges, with a regression test covering the SoundCue node chain.
- `#3-verify-fix` `DONE` tester — Verified: fresh `asset.dump` on `/App/Audio/Cue_Drone` emits wave_player metadata `duration: 19.022993, numChannels: 2, sampleRate: 44100, bLooping: true`. Broader checks: `Cue_AssemblyLine` scir.txt emits mixer `InputVolume`, delay `DelayMin/DelayMax`, looping/wave_player tree; `Cue_Reservoir` emits modulator `PitchMin/PitchMax/VolumeMin/VolumeMax`. No project SoundCues contain Attenuation or Branch nodes, so those node types weren't exercised against real assets, but all four metadata categories the ticket called out for project sample (wave audio + modulator) emit correctly.
