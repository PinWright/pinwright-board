---
id: B-level-sequence-missing-subsequences
title: "level_sequence.json doesn't list nested sub-sequence references"
status: DONE
severity: Low
category: bug
tags: [level-sequence, sidecar]
---

# level_sequence.json doesn't list nested sub-sequence references

`level_sequence.json` captures tracks, bindings, frame rates, and camera-cut track, but doesn't surface nested ULevelSequence references (via `FMovieSceneSubSection` inside MovieSceneSubTrack tracks) as a top-level dependency list.

Result: a sequence's full dependency graph isn't visible without walking individual tracks.

## Sample

`LevelSequenceDumpBuilder.cpp` iterates `Sequence->GetMovieScene()->GetTracks()` generically but doesn't special-case `MovieSceneSubSection` references. None currently visible in the dump tree (no samples present), but schema gap is real.

## Fix sketch

In `LevelSequenceDumpBuilder.cpp`, after bindings: walk all tracks, find `UMovieSceneSubSection` instances, collect their `GetSequence()->GetPathName()` into a deduplicated `subSequences` array at the dump root.

## History
- `#1-subseq-missing` `OPEN` reporter — schema completeness for dependency analysis.
- `#2-subseq-root-list` `IN-REVIEW` developer — Added deterministic root `subSequences` emission in `LevelSequenceDumpBuilder.cpp` by walking direct movie-scene track sections, collecting non-null `UMovieSceneSubSection::GetSequence()` paths, deduplicating, and sorting before JSON output. Existing nested `innerSequencePath` section output remains unchanged.
- `#3-verify-subseq-emission` `DONE` tester — Verified: `asset.dump` on `/Game/ArchvisProject/Sequences/Cameras/Seq_Master_Cameras` produced `level_sequence.json` with `subSequences: ["/Game/ArchvisProject/Sequences/Cameras/Seq_CineCameraActor01.Seq_CineCameraActor01", "Seq_CineCameraActor02...", "Seq_CineCameraActor03..."]` — three entries, alphabetically sorted, deduplicated, matching the three `MovieSceneCinematicShotTrack` sections. Control dumps (`LS_DroneRotation`, `LS_DroneBG`, `LS_DroneBG_V2`) correctly emit `subSequences: []` because their MovieScene tracks contain no `UMovieSceneSubSection` instances.
