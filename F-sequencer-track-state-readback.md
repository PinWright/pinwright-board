---
id: F-sequencer-track-state-readback
title: "No read-back for sequencer track mute / solo / lock state — set_track_* writes are unverifiable"
status: IN-REVIEW
severity: High
category: feature
tags: [sequencer, readback, parity, round-trip]
---

# No read-back for sequencer track mute / solo / lock state — `set_track_*` writes are unverifiable

The sequencer namespace has three write RPCs that mutate per-track playback state:

- `sequencer.set_track_muted` → `Track->SetEvalDisabled(bMuted)` (`SequenceHandler.cpp:2025`)
- `sequencer.set_track_solo` → `Track->SetEvalDisabled(Track != SoloTrack)` on every track (`SequenceHandler.cpp:2099-2105`); solo is *simulated* by muting all other tracks
- `sequencer.set_track_locked` → `Section->SetIsLocked(bLocked)` for each of the track's sections (`SequenceHandler.cpp:2182-2186`)

These writes are real and persisted (`bIsEvalDisabled` on `UMovieSceneTrack`, `bIsLocked` on `UMovieSceneSection`), but **no read RPC in the namespace surfaces that state**, so a caller cannot verify the result, cannot read state a designer set in the Sequencer UI, and cannot make `set_track_solo` idempotent (it has no way to know which track is currently soloed). The round-trip is unverifiable through the MCP.

Confirmed against every plausible read surface:

- `sequencer.list_tracks` emits only `trackName`, `trackType`, `displayName`, `isMasterTrack`, `sectionCount`, `bindingName`, `bindingGuid` per track (`SequenceHandler.cpp:2645-2651`, `:2670-2677`) — no `muted` / `isEvalDisabled`. After soloing+locking one track and muting another, the post-write `list_tracks` output is **byte-identical** to the pre-write output.
- `sequencer.list_sections` → `SequenceHelpers::BuildListedSectionJson` (`SequenceHandler.cpp:149-164`) wraps `MovieSceneJsonUtils::BuildSectionJson` (`Utils/MovieSceneJsonUtils.h:74-111`), which emits `range`, `blendType`, `rowIndex`, `channels` (+ sub-section fields) — never `Section->IsLocked()`.
- `sequencer.get_metadata` / `sequencer.get_properties` cover asset metadata and playback range / frame rate only — no per-track flags.
- `asset.dump` (`LevelSequenceDumpBuilder.cpp`) and `sequencer.get_camera_cut_track` both go through `MovieSceneJsonUtils::BuildTrackJson` (`Utils/MovieSceneJsonUtils.h:113-133`), which likewise omits `IsEvalDisabled()`. So even the dump baseline can't see mute/solo state.

Note: `F-sequencer-visibility-track` (line 30) asserts "Round-trip coverage for the rest of the surface is already in place: `sequencer.list_sections` returns the authored section state, and `sequencer.set_track_locked` / `set_track_muted` / `set_track_solo` handle the per-track flags." That is half-true — the setters exist, but there is no matching reader, so the round-trip claim does not hold for these three flags.

**Fix:** Surface the state in the existing read shapes (preferred — no new method, byte-stays-aligned with the dump policy):
1. Add `isEvalDisabled` (bool, from `Track->IsEvalDisabled()`) to the per-track object in `sequencer.list_tracks` and to `MovieSceneJsonUtils::BuildTrackJson` (so `asset.dump` and `get_camera_cut_track` gain it too). Surfacing `isEvalDisabled` covers both mute and the simulated-solo result; an additional derived `muted` alias may be emitted for symmetry with `set_track_muted`.
2. Add `isLocked` (bool, from `Section->IsLocked()`) to `MovieSceneJsonUtils::BuildSectionJson` so it appears in `sequencer.list_sections` (and the dump). A track is "locked" when all its sections are locked; callers can derive that, or `list_tracks` can additionally emit an aggregate `allSectionsLocked`.

Because true native solo state doesn't exist in Unreal (solo is simulated via per-track eval-disable), there is no separate "solo" flag to read back — `isEvalDisabled` per track is the honest representation and should be documented as such.

**Workaround:** drop to `python.execute` and read `Track.is_eval_disabled()` / `Section.is_locked()` directly, or `property.get` the `bIsEvalDisabled` / `bIsLocked` UPROPERTYs — i.e. the dedicated sequencer read surface is bypassed entirely.

## Repro (verbatim, replayed live on the shipped Content Examples asset)

Path: `/Game/ExampleContent/Sequencer/LevelSequences/4_3_ControlRigTracks_LS.4_3_ControlRigTracks_LS`

1. `sequencer.list_tracks {path}` → 3 tracks, each `{trackName,trackType,displayName,isMasterTrack,bindingName,bindingGuid,sectionCount}` — no state fields.
2. `sequencer.set_track_solo {path, trackName:"MovieSceneControlRigParameterTrack_3", solo:true}` → `{"trackName":"MovieSceneControlRigParameterTrack_3","solo":true,"note":"Solo is simulated by muting all other tracks..."}`
3. `sequencer.set_track_locked {path, trackName:"MovieSceneControlRigParameterTrack_3", locked:true}` → `{"trackName":"...","locked":true}`
4. `sequencer.set_track_muted {path, trackName:"MovieSceneSkeletalAnimationTrack_0", muted:true}` → `{"trackName":"...","muted":true}`
5. `sequencer.list_tracks {path}` (readback) → **byte-identical to step 1** — the solo, lock, and mute just applied are invisible.
6. `sequencer.list_sections {path, trackName:"MovieSceneControlRigParameterTrack_3"}` → `{"sections":[],"sectionCount":0}` (filter quirk aside, `BuildListedSectionJson` has no `isLocked` field regardless).

## History
- `#1-initial-repro` `OPEN` reporter — `set_track_muted`/`set_track_solo`/`set_track_locked` write real persisted state (`SetEvalDisabled` / `SetIsLocked`) but no sequencer read RPC exposes it: `list_tracks` (`SequenceHandler.cpp:2645-2651`) and `BuildTrackJson` (`MovieSceneJsonUtils.h:113-133`) omit `IsEvalDisabled`; `BuildSectionJson` (`MovieSceneJsonUtils.h:74-111`) and `BuildListedSectionJson` (`SequenceHandler.cpp:149-164`) omit `IsLocked`; `get_metadata`/`get_properties` cover other state. Replayed live on `4_3_ControlRigTracks_LS`: post-write `list_tracks` is byte-identical to pre-write, confirming the round-trip is unverifiable via the MCP.
- `#2-readback-fields` `IN-REVIEW` developer — Surfaced the persisted state through the existing read shapes (no new RPC). `MovieSceneJsonUtils::BuildSectionJson` now emits `isLocked` (`Section->IsLocked()`) and `BuildTrackJson` now emits `isEvalDisabled` (`Track->IsEvalDisabled()`), so `sequencer.list_sections`, `asset.dump` (`level_sequence.json`), and `sequencer.get_camera_cut_track` all gain the lock/mute-solo state (`Utils/MovieSceneJsonUtils.h`). `sequencer.list_tracks` per-track objects (both master and binding loops) gained `isEvalDisabled` plus an aggregate `allSectionsLocked` (new `SequenceHelpers::AreAllSectionsLocked` — a track reads locked only when every section is locked; empty tracks report false) (`Private/Handlers/Sequencer/SequenceHandler.cpp`). `isEvalDisabled` is the honest representation of both mute and the simulated solo (Unreal has no native solo flag), as the ticket framed. Bumped the `level_sequence.json` asset-dump aspect version 2→3 (`Private/Handlers/Asset/AssetDumpCache.cpp`) so stale `tree.xml`/dump caches regenerate with the new fields. Regression test `Private/Tests/Sequencer/TestTrackStateReadback.cpp` (`EditorAutomationRpcGateway.Sequencer.TrackStateReadback.ListTracksReflectsSetters`) drives the production handlers end-to-end: builds a transient sequence + track via `add_level_visibility_track`, asserts baseline `list_tracks` carries `isEvalDisabled=false`/`allSectionsLocked=false`, mutates via `set_track_muted`+`set_track_locked`, then asserts `list_tracks` reports both true — fails if the readback fields are reverted.
