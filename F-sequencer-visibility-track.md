---
id: F-sequencer-visibility-track
title: "Add Sequencer visibility-track support (MovieSceneLevelVisibilityTrack)"
status: DONE
severity: Medium
category: feature
tags: [sequencer, level-visibility, sublevel]
---

# Add Sequencer visibility-track support (MovieSceneLevelVisibilityTrack)

`UMovieSceneLevelVisibilityTrack` is the standard way to drive sublevel visibility from a `ULevelSequence` — each `UMovieSceneLevelVisibilitySection` carries a list of streaming-level names and a `bVisible` flag (`EMovieSceneLevelVisibility::Visible` / `Hidden`) over a frame range. This is how cinematics swap between gameplay and cinematic sublevels, and how WLS-based projects toggle streaming layers per shot.

No handler in `Private/Handlers/Sequencer/` authors this track type. Grep of `visibility_track`, `MovieSceneLevelVisibility`, and `LevelVisibilitySection` across `docs/rpc-method-reference.generated.md` and `Private/Handlers/Sequencer/` returned zero matches. The generic `sequencer.add_track` path can in principle construct a `UMovieSceneLevelVisibilityTrack` from its raw class string, but it cannot populate the section's `LevelNames` array or `Visibility` enum — those are section-level UPROPERTY fields with no live setter exposed. Callers are blocked from authoring sublevel-visibility cinematics through the MCP entirely.

**Fix:** Add `sequencer.add_level_visibility_track` registered alongside `sequencer.add_track` in `SequencerHandler.cpp`. Shape:

- Required `sequencePath` (level-sequence asset path).
- Required `levelNames` (array of FName strings — short package names of streaming sublevels, matching how the editor stores them in the section).
- Required `bVisible` (bool — true → `EMovieSceneLevelVisibility::Visible`, false → `Hidden`).
- Optional `range` (`{start, end}` frame numbers; defaults to the sequence's full playback range).
- Optional `rowIndex` (defaults to 0).
- Optional `reuseTrack` (bool, default true — append a new section to the existing master track instead of creating a duplicate track).

The handler should:
1. Load the sequence and resolve `MovieScene->FindMasterTrack<UMovieSceneLevelVisibilityTrack>()`; create one via `AddMasterTrack` if missing or `reuseTrack=false`.
2. Call `Track->CreateNewSection()` (or `NewObject<UMovieSceneLevelVisibilitySection>` + `Track->AddSection`), set `LevelNames`, `Visibility`, `SetRange`, and `SetRowIndex`.
3. Mark sequence dirty, return `{ sequencePath, track: <BuildTrackJson>, section: <BuildSectionJson> }` using the shared helpers in `Utils/MovieSceneJsonUtils.h` (extracted in `F-rpc-sequencer-list-sections`) so the response is byte-identical to `list_tracks` / `list_sections` / `level_sequence.json`.

Round-trip coverage for the rest of the surface is already in place: `sequencer.list_sections` returns the authored section state, and `sequencer.set_track_locked` / `set_track_muted` / `set_track_solo` handle the per-track flags. Section-level edits (changing `LevelNames` or flipping `Visibility` after creation) are out of scope for this ticket — defer to a follow-up `sequencer.set_level_visibility_section` if a concrete consumer surfaces.

Build dependency: `UMovieSceneLevelVisibilityTrack` lives in the `LevelSequence` module, which is already referenced by existing sequencer handlers — no new module add to `EditorAutomationRpcGateway.Build.cs` expected.

## History
- `#1-initial-repro` `OPEN` reporter — No handler authors `MovieSceneLevelVisibilityTrack`/`UMovieSceneLevelVisibilitySection` anywhere in `Private/Handlers/Sequencer/` (`SequenceHandler.cpp`, `SequencerHandler.cpp`) or in `docs/rpc-method-reference.generated.md`. Generic `sequencer.add_track` can construct the track but cannot populate the section's `LevelNames` / `Visibility` fields, blocking sublevel-visibility cinematic authoring through MCP. Proposed handler shape, helper reuse (`MovieSceneJsonUtils.h` `BuildTrackJson`/`BuildSectionJson`), and out-of-scope follow-ups documented above.
- `#2-implement-visibility-track` `IN-REVIEW` developer — Added `sequencer.add_level_visibility_track` in `SequencerHandler.cpp`. Handler resolves or creates a `UMovieSceneLevelVisibilityTrack`, creates a `UMovieSceneLevelVisibilitySection`, sets `LevelNames`/`Visibility`/`SetRange`/`SetRowIndex`. Response shaped via shared `BuildTrackJson`/`BuildSectionJson` helpers. UE 5.6 API correction vs ticket: enum is `ELevelVisibility` (not `EMovieSceneLevelVisibility`); `MovieScene::AddTrack<T>()`/`FindTrack<T>()` no-arg overloads replace the renamed `AddMasterTrack`/`FindMasterTrack`. No Build.cs change (module `MovieSceneTracks` already linked). Tests appended at `Tests/Sequencer/TestAddLevelVisibilityTrack.cpp`.
- `#3-verify-fix` `DONE` tester — Verified: `sequencer.add_level_visibility_track?` returns the documented schema (sequencePath/levelNames/bVisible required; range/rowIndex/reuseTrack optional). Live call on `/Game/NewLevelSequence.NewLevelSequence` with `levelNames=["L_TestSubLevel"], bVisible=true, reuseTrack=true` returned `success:true`, `track.class:"MovieSceneLevelVisibilityTrack"`, `sectionCount:1`, `reused:false` (no prior track). Follow-up `sequencer.list_sections` confirms the persisted section with `trackClass:"MovieSceneLevelVisibilityTrack"` and the full-playback range `{0, 48000}`.
