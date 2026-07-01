---
id: F-rpc-audio-describe-sound-cue
title: "Add live RPC `audio.authoring.describe_sound_cue` for full SoundCue graph"
status: DONE
severity: High
category: feature
tags: [audio, sound-cue, dump-parity, asset-dump]
---

# Add live RPC `audio.authoring.describe_sound_cue` for full SoundCue graph

`SoundCueDumpBuilder::BuildSoundCueJson` (Source/.../Handlers/Asset/SoundCueDumpBuilder.cpp) emits the entire cue graph: `firstNode` path, an ordered `nodes[]` array (DFS from FirstNode plus orphan AllNodes pass), and per-node `className`, sparse-diff `properties` against the class CDO, `edges[]` (child path or null), and `soundWave` resolved through `USoundNodeWavePlayer::GetSoundWave()` with a private `SoundWaveAssetPtr` reflective fallback. Orphan nodes are flagged via `"orphaned": true`.

Live audio RPCs cover none of this. `audio.authoring.get_audio_info` (AudioAuthoringHandler.cpp:2302) returns only `duration`, `nodeCount` (from `AllNodes.Num()`), and `attenuationPath` for SoundCue assets. Every other audio handler is a writer (`create_sound_cue`, `add_cue_node`, `connect_cue_nodes`, `set_cue_attenuation`, etc.) or a runtime playback op (AudioHandler.cpp). There is no RPC anywhere that exposes the cue's node graph, edges, per-node settings, wave references, or orphan state. Confirmed by grepping `Source/` for `cue_node`, `cue_graph`, `describe_sound_cue`, `FirstNode`, `ChildNodes`, and the surrounding audio inspect verbs — no live reader exists.

This violates the asset-dump-parity policy (anything a dump emits must also be reachable via a live MCP RPC). It also blocks every inspect-after-mutate workflow: an agent that runs `add_cue_node` or `connect_cue_nodes` cannot read back the resulting graph without re-running `asset.dump_folder` and reading sidecars, which is slow and stale by the time the next mutation lands.

**Fix:** Add `audio.authoring.describe_sound_cue` (params: `assetPath`) that resolves the cue via the existing `LoadSoundCueFromPath()` helper, then delegates to `SoundCueDumpBuilder::BuildSoundCueJson(Cue)` and ships the result. The builder is already DLL-exported (`EDITORAUTOMATIONRPCGATEWAY_API` in SoundCueDumpBuilder.h), so the handler is a thin wrapper — same shape on both surfaces, no schema drift, no duplicated traversal logic. Wire it next to `get_audio_info` in AudioAuthoringHandler.cpp; mention it on the audio.authoring wiki page's "Inspect-after-mutate" line.

## History
- `#1-initial-repro` `OPEN` reporter — SoundCue dump sidecar (firstNode + nodes[] with className/properties/edges/soundWave/orphan flag, produced by SoundCueDumpBuilder::BuildSoundCueJson) has no live-RPC equivalent. `audio.authoring.get_audio_info` only returns duration/nodeCount/attenuationPath; all other audio handlers are writers or runtime playback. Verified via grep on `Source/` for SoundCue, cue_node, cue_graph, FirstNode, ChildNodes — no live graph reader exists. Violates asset-dump parity policy and blocks inspect-after-mutate workflows.
- `#2-describe-sound-cue-rpc` `IN-REVIEW` developer — Added `audio.authoring.describe_sound_cue` as a read-only live RPC that loads SoundCue assets through the existing audio loader and delegates graph JSON to `SoundCueDumpBuilder::BuildSoundCueJson`; updated audio.authoring wiki inspect guidance and added a focused handler regression test asserting SoundCue graph shape. This closes the live-RPC parity gap without changing `get_audio_info`.
- `#3-verify-fix` `DONE` tester — Verified: wiki page for `audio.authoring.describe_sound_cue` exists with `assetPath` param; called it on `/Game/Audio/FPV_SOUND/UI/UISFX_Start_Cue.UISFX_Start_Cue` and got expected payload — `assetKind:"SoundCue"`, `firstNode` set to `SoundNodeMixer_0`, ordered `nodes[]` (5 entries) each with `className`, sparse `properties` (diff vs CDO), `edges[]`, and `soundWave` resolved on WavePlayer nodes via `SoundWaveAssetPtr`. Matches the sidecar shape described in the ticket.
