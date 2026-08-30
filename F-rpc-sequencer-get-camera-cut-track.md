---
id: F-rpc-sequencer-get-camera-cut-track
title: "Add live RPC for camera-cut track (parity with `level_sequence.json#cameraCutTrack`)"
status: DONE
severity: Medium
category: feature
tags: [sequencer, asset-dump, parity, policy]
---

# Add live RPC for camera-cut track (parity with `level_sequence.json#cameraCutTrack`)

`LevelSequenceDumpBuilder::BuildLevelSequenceJson` (`Source/PinWright/Private/Handlers/Asset/LevelSequenceDumpBuilder.cpp:91-98`) emits a top-level `cameraCutTrack` field — the full track object (name, class, sectionCount, sections with range/blendType/rowIndex/channels) when `MovieScene->GetCameraCutTrack()` returns non-null, or an explicit JSON `null` otherwise.

No live RPC surfaces this. `sequencer.list_tracks` (`SequenceHandler.cpp:2379`) iterates only `MovieScene->GetTracks()` and per-binding `Binding.GetTracks()` — the camera-cut track lives on a separate slot accessed via `MovieScene->GetCameraCutTrack()`/`AddCameraCutTrack()` and is not enumerated by either of those calls. The camera-cut track is therefore invisible to every existing `sequencer.*` reader despite being a first-class fixture in the dump.

This violates the "asset dumps must not have exclusive functionality" policy: a caller that authored a camera-cut track via `sequencer.add_camera_track` (`SequencerHandler.cpp:255`) cannot read back its presence, sections, or per-section state without re-dumping the asset.

**Fix:** Two viable shapes; pick one when implementing:

1. **Dedicated reader `sequencer.get_camera_cut_track`** — required `path`; returns `{ sequencePath, cameraCutTrack: <track-or-null> }` with the same shape as the dump's `cameraCutTrack` field. Thin and explicit; mirrors the dedicated slot in `UMovieScene`.
2. **Extend `sequencer.list_tracks`** to include the camera-cut track in its `tracks` array (with a discriminator like `isCameraCutTrack: true` and `isMasterTrack: false`/`true` per existing convention). Then per-section state is automatically covered by `F-rpc-sequencer-list-sections` once that lands. Cheapest in surface area but mixes a singleton slot into a list.

Recommend (1) for an unambiguous "does this sequence have a camera-cut track?" answer plus full payload in one call, and consider (2) as a follow-up so generic track enumerators don't miss it. Either way, reuse the same `BuildTrackJson`/`BuildSectionJson`/`BuildChannelEntriesJson`/`MakeFrameRangeObject`/`BlendTypeToString` helpers that `F-rpc-sequencer-list-sections` proposes extracting from `LevelSequenceDumpBuilder.cpp`'s anonymous namespace into `Utils/MovieSceneJsonUtils.h`, so the live RPC and the dump emit byte-identical payloads.

See also: `F-rpc-sequencer-list-sections` (per-section range/blendType/rowIndex/channels parity for tracks already enumerated by `list_tracks`).

## History
- `#1-initial-repro` `OPEN` reporter — `LevelSequenceDumpBuilder` emits `cameraCutTrack` (full track object or null) at `LevelSequenceDumpBuilder.cpp:129-136`. `sequencer.list_tracks` iterates `MovieScene->GetTracks()` and per-binding tracks only — camera-cut track lives on a separate `GetCameraCutTrack()` slot and is omitted. Verified no other `sequencer.*` reader (32 enumerated) returns camera-cut track state. Dump-exclusive functionality, violates the parity policy.
- `#2-added-camera-cut-reader` `IN-REVIEW` developer — Added `sequencer.get_camera_cut_track`, shared MovieScene track JSON serialization helpers with `LevelSequenceDumpBuilder`, and covered camera-cut track response shape with an automation test.
- `#3-verify-fix` `DONE` tester — Verified: wiki page exists with required `path` param; `sequencer.get_camera_cut_track` on `/App/Sequences/FlythroughSequence01` returned `{sequencePath, cameraCutTrack:{name:"None", class:"MovieSceneCameraCutTrack", sectionCount:8, sections:[8x {range:{start,end}, blendType:"Absolute", rowIndex:0, channels:[]}]}}` — byte-identical to `level_sequence.json#cameraCutTrack` in the cached dump. Null case on `/Game/NewLevelSequence` returned `cameraCutTrack: null` as specified.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `:129-136` had drifted onto per-binding name/track assembly inside the bindings loop; the `cameraCutTrack` if/else the body quotes is verbatim-correct at `:91-98`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
