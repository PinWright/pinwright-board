---
id: F-sequencer-sub-sequences
title: "Add sub-sequence track + section authoring (`sequencer.add_sub_sequence`, `sequencer.set_sub_section_range`)"
status: DONE
severity: Medium
category: feature
tags: [sequencer, cinematics, authoring, parity]
---

# Add sub-sequence track + section authoring

`UMovieSceneSubTrack` / `UMovieSceneSubSection` are not authorable through any
registered `sequencer.*` handler. Grepping `Source/PinWright/Private/Handlers/Sequencer/`
turns up zero references to `SubTrack`, `SubSection`, `sub_sequence`, or
`SubSequence`; the public catalog in `docs/rpc-method-reference.generated.md`
enumerates 38 `sequencer.*` methods (lines 9303–9643) and none of them surface
sub-sequence authoring. `sequencer.add_track` cannot stand in: sub-sections
require an inner `ULevelSequence` reference and per-section start/duration
configured at creation, which a generic `add_track` cannot express.

This blocks shot-based cinematic workflows and re-use of shared sequence
fragments (intro stingers, looped ambient beats, master cinematics composed of
per-shot sequences). Today a caller can build only flat sequences.

**Fix:** Add two handlers in
`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`
alongside `sequencer.add_track`:

1. `sequencer.add_sub_sequence` — required `path` (parent sequence asset),
   `innerSequencePath` (child sequence asset), `startFrame` (int), `durationFrames`
   (int); optional `rowIndex` (int, default 0), `timeScale` (float, default 1.0).
   Resolves or creates the parent's `UMovieSceneSubTrack`, calls
   `Track->AddSequence(InnerSequence, StartFrame, DurationFrames)`, applies
   `rowIndex` and `Parameters.TimeScale`, returns `{ sectionGuid, trackName,
   rowIndex, range:{start,end} }`.
2. `sequencer.set_sub_section_range` — required `path`, `sectionGuid`, `startFrame`,
   `durationFrames`; optional `timeScale`. Looks up the section by GUID on the
   sub-track and mutates its range/time-scale in place.

**Parity follow-up:** Extend `sequencer.list_sections` (DONE in
`F-rpc-sequencer-list-sections`) so sub-section rows surface
`innerSequencePath` and `timeScale` alongside the existing `range`/`blendType`/
`rowIndex`/`channels` payload — without that, callers can author sub-sections
but cannot read them back, re-introducing the dump-exclusivity gap that ticket
closed. Reuse the shared `Utils/MovieSceneJsonUtils.h` helpers so the live RPC
and `level_sequence.json` stay byte-identical.

See also: `F-rpc-sequencer-list-sections`, `F-rpc-sequencer-get-camera-cut-track`.

## History
- `#1-initial-repro` `OPEN` reporter — Verified no handler references `MovieSceneSubTrack`/`MovieSceneSubSection`/`sub_sequence` under `Private/Handlers/Sequencer/`. Public RPC catalog enumerates 38 `sequencer.*` methods with no sub-sequence authoring path. Blocks shot-based cinematics and shared-fragment re-use.
- `#2-implement-sub-sequences` `IN-REVIEW` developer — Added `sequencer.add_sub_sequence` and `sequencer.set_sub_section_range` in `SequenceHandler.cpp`. Locates or creates `UMovieSceneSubTrack`, calls `Track->AddSequenceOnRow(InnerSequence, FFrameNumber(start), durationFrames, rowIndex)`, applies `Section->Parameters.TimeScale` when provided. Section identity uses `UMovieSceneSection::GetSignature()` (FGuid stable per section). Extended `MovieSceneJsonUtils::BuildSectionJson` with a `Cast<UMovieSceneSubSection>` branch that appends `innerSequencePath` and (when `TimeScale.GetType() == FixedPlayRate`) `timeScale`, so `list_sections` and asset dumps surface sub-section state. Three dispatcher tests appended to `Tests/Media/TestSequencerHandlers.cpp`. No Build.cs change — `MovieScene` module already a transitive dependency.
- `#3-verify-fix` `DONE` tester — Verified: schemas for both new RPCs are registered (`sequencer.add_sub_sequence?`, `sequencer.set_sub_section_range?` return full param specs). End-to-end on `/Game/NewLevelSequence` with inner `/Game/Meadow_Environment_Set/Maps/LS_DroneRotation`: `add_sub_sequence` returned `sectionGuid=932F537E…`, `rowIndex=2`, range `{0,1000}`, and the embedded section JSON included `innerSequencePath` + `timeScale=1.5`. `set_sub_section_range` with that GUID mutated to range `{250,750}` and `timeScale=2.0`. Final `list_sections` confirmed a third section with `trackClass=MovieSceneSubTrack`, `innerSequencePath=…/LS_DroneRotation.LS_DroneRotation`, `timeScale=2`, range `{250,750}`. Used `NewLevelSequence` because `sequencer.create` on `/Game/App/UI/Test/` was timing out under unrelated editor-load contention; mutation left in place since NewLevelSequence is a transient scratch asset.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 2 body citations repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
