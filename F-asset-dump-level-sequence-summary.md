---
id: F-asset-dump-level-sequence-summary
title: "Level Sequence summary sidecar (level_sequence.json)"
status: DONE
severity: Medium
category: feature
tags: []
---

# Level Sequence summary sidecar (level_sequence.json)

`ULevelSequence` assets dumped via `asset.dump` currently land on the catchall branch and
emit only `meta.json` plus generic `properties.json`. Property reflection exposes object
references but no track / section / key summary, so dump consumers cannot see what the
sequence actually does without round-tripping back to the editor.

Add a `level_sequence.json` sidecar with a compact, stable summary of the sequence's
`UMovieScene`: tracks (name, class), sections (range, blend type), and keys (count and
type per channel). Format must be diff-stable and LLM-facing, mirroring the style of
the StaticMesh / Texture2D / SoundWave summaries shipped under the parent ticket.

This is split off from `F-asset-dump-native-summary-aspects` because walking
`UMovieScene` + `UMovieSceneTrack` + `UMovieSceneSection` (and per-channel key data)
is materially heavier scope than the per-class field reads the parent ticket covers.

**Workaround:** Inspect the sequence in the editor or call type-specific live RPCs.
**Fix:** Add a `LevelSequenceDumpBuilder` under `Private/Handlers/Asset/`, dispatch
from `AssetDumpHandler::BuildAllFilesForAsset`, register `level_sequence.json` in
`DumpFileNames` and `Canonical[]`. Use `MovieSceneToolsModule` track editors only if
needed for human-readable section labels.

## History
- `#1-split-from-native-summary-aspects` `OPEN` developer — Split off from F-asset-dump-native-summary-aspects implementation (Sprint 2026-05-09). Track/section/key summary requires UMovieScene + MovieSceneTracks walking and is materially heavier scope than StaticMesh/Texture2D/SoundWave.
- `#2-level-sequence-builder` `IN-REVIEW` developer — Added LevelSequenceDumpBuilder.{h,cpp} under Private/Handlers/Asset, registered DumpFileNames::LevelSequence and dispatched from AssetDumpHandler::BuildAllFilesForAsset default tail; ULevelSequence assets now emit level_sequence.json (tickResolution/displayRate/playbackRange, binding/spawnable/possessable counts, tracks with sections+channels, cameraCutTrack, sorted bindings); regression test FAssetDumpLevelSequenceSummaryTest in TestAssetDumpNativeSummaries.cpp.
- `#3-verify-fix` `DONE` tester — Verified: asset.dump on /App/Sequences/FlythroughSequence01 emitted level_sequence.json alongside meta/properties; sidecar contains tickResolution (24000/1), displayRate (30/1), playbackRange (-4800..2720000), bindingCount=8/spawnableCount=0/possessableCount=8, GUID-sorted bindings with MovieScene3DTransformTrack sections+channels, cameraCutTrack with 8 sections, top-level tracks array — all fields from the IN-REVIEW claim present.
