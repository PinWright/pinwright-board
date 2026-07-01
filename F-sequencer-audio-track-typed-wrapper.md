---
id: F-sequencer-audio-track-typed-wrapper
title: "Typed sequencer.add_audio_track wrapper"
status: DONE
severity: Low
category: feature
tags: [sequencer, audio, level-sequence]
---

# Typed sequencer.add_audio_track wrapper

`UMovieSceneAudioTrack` / `UMovieSceneAudioSection` are the standard mechanism for placing `USoundBase` assets (sound waves, cues, MetaSounds) on a `ULevelSequence`. They are used for both master audio tracks (sequence-level music/SFX) and binding-attached audio tracks (per-actor dialogue, footsteps, etc.), and expose per-section volume and pitch multipliers plus a start-time offset into the sound asset.

`SequenceHandler.cpp:81` already includes `Tracks/MovieSceneAudioTrack.h`, but grep of `audio_track`, `MovieSceneAudioTrack`, and `add_audio_track` across `docs/rpc-method-reference.generated.md` and `Private/Handlers/Sequencer/` returns zero handler-side matches — the include is unused. Callers wanting to author audio on a sequence must fall back to the generic `sequencer.add_track` + `sequencer.add_section` path, which can construct the track but cannot populate the section's `Sound`, `StartFrameOffset`, `SoundVolume`, or `PitchMultiplier` UPROPERTY fields (no live setters are exposed). Audio cinematic authoring through MCP is effectively blocked.

**Fix:** Add `sequencer.add_audio_track` to `SequencerHandler.cpp`, mirroring the existing `add_animation_track` / `add_camera_track` / `add_transform_track` wrappers immediately above. Shape (matching project param conventions — `startTime` in seconds, not raw frames, like `add_animation_track`):

- Required `sequencePath` (level-sequence asset path).
- Optional `bindingGuid` (object binding GUID — when present, attach the track to the binding via `MovieScene->AddTrack<UMovieSceneAudioTrack>(BindingGuid)`; when omitted/empty, create as a master track via `MovieScene->AddMasterTrack<UMovieSceneAudioTrack>()`).
- Required `soundPath` (path to a `USoundBase` asset — accepts `USoundWave`, `USoundCue`, MetaSound source, etc.).
- Optional `startTime` (seconds, default 0).
- Optional `duration` (seconds; default = `SoundBase->GetDuration()`, with a sanity fallback for looping/streaming sounds whose duration is `INDEFINITELY_LOOPING_DURATION`).
- Optional `volume` (float, default 1.0 → `AudioSection->SetSoundVolume(...)` if a live setter exists, else write via the section's volume channel default).
- Optional `pitch` (float, default 1.0 → same approach via pitch channel).
- Optional `rowIndex` (int, default 0).

The handler should follow the `add_animation_track` template: load the sequence, validate the MovieScene, parse `bindingGuid` if provided, load the `USoundBase` via `LoadObject<USoundBase>`, create the track (binding-attached vs master based on guid presence), `CreateNewSection()` → cast to `UMovieSceneAudioSection`, `AddSection`, set `Sound`, `SetRange` from display-rate-converted frames, mark the MovieScene dirty, return a response object with `success`, `bindingGuid` (or `master: true`), `soundPath`, `startTime`, `duration`, plus the standard `AddAssetVerification(Resp, LevelSequence)` block.

Build dependency: `UMovieSceneAudioTrack` / `UMovieSceneAudioSection` live in the `MovieSceneTracks` module, already referenced by sibling sequencer handlers — no `Build.cs` change expected.

Out of scope: editing volume/pitch curves over time (sectional channel keyframing), audio attenuation overrides, and post-creation section edits — defer to follow-ups if a concrete consumer surfaces.

## History
- `#1-initial-repro` `OPEN` reporter — `SequenceHandler.cpp:81` includes `Tracks/MovieSceneAudioTrack.h` but no handler in `Private/Handlers/Sequencer/` authors `UMovieSceneAudioTrack` / `UMovieSceneAudioSection`; zero matches for `audio_track` / `MovieSceneAudioTrack` / `add_audio_track` in `docs/rpc-method-reference.generated.md`. Callers blocked from authoring audio cinematics because generic `add_track` + `add_section` cannot populate `Sound`/volume/pitch fields. Proposed handler shape mirrors the existing `add_animation_track` / `add_camera_track` / `add_transform_track` wrappers (seconds-based `startTime`, master-vs-binding decided by `bindingGuid` presence, response uses `AddAssetVerification` helper).
- `#2-implement-audio-track` `IN-REVIEW` developer — Added `sequencer.add_audio_track` in `SequencerHandler.cpp` mirroring the `add_animation_track` template. Master-vs-binding decided by `bindingGuid` presence (UE 5.6 uses `MovieScene::AddTrack<T>()` no-arg overload, not `AddMasterTrack` as ticket suggested). Loads `USoundBase`, creates `UMovieSceneAudioSection`, sets `Sound`/range/rowIndex. Volume/pitch authored as channel defaults via `Section->GetChannelProxy().GetChannel<FMovieSceneFloatChannel>(0/1)->SetDefault(...)` because the section exposes no public setters. Duration fallback for `INDEFINITELY_LOOPING_DURATION` (10000.0f from `AudioDefines.h`) defaults to 1.0s. Two dispatcher tests appended to `Tests/Media/TestSequencerHandlers.cpp`.
- `#3-verify-fix` `DONE` tester — Verified: `sequencer.add_audio_track?` exposes the full schema (sequencePath/soundPath required; bindingGuid/startTime/duration/volume/pitch/rowIndex optional). Live call against `/Game/NewLevelSequence` with `soundPath=/Game/Audio/FPV_SOUND/UI/SFX_Gate_sound_1a`, `startTime=0.5`, `duration=2.0`, `volume=0.75`, `pitch=1.25` returned `success:true`, `master:true`, track class `MovieSceneAudioTrack`, one section with range 15→75 (matches 0.5s+2.0s at default 30fps tick conversion) and 3 channels (ActorReferenceData + 2 FloatChannels for volume/pitch).
