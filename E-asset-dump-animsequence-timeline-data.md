---
id: E-asset-dump-animsequence-timeline-data
title: "AnimSequence dumps lack timeline/notify/curve sidecar"
status: DONE
severity: Medium
category: ergonomic
tags: [asset-dump, anim-sequence, sidecar, coverage]
---

# AnimSequence dumps lack timeline/notify/curve sidecar

854 `AnimSequence` dumps under `.editor-automation/asset-dumps/` emit only
`meta.json` and `properties.json`. There is no type-specific sidecar carrying
the timeline data an LLM agent actually wants (animation length, frame rate,
notifies with timing, curves with key counts, sync markers, additive type,
skeleton reference) as a compact, contract-stable shape.

`AssetDumpHandler::BuildAllFilesForAsset` has no `UAnimSequence` branch
(grep for `AnimSequence` / `AnimMontage` in
`Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp`
returns zero hits). The asset falls through to the generic class-property
walk, so the only timeline data present is whatever the reflected `UProperty`
set surfaces.

What IS present in `properties.json` today (sample:
`App/Meshes/Clutch/SK_Clutch02_Anim/properties.json`, ~5 KB total):

- `SequenceLength`: `33.299999237060547` (float)
- `NumFrames` / `NumberOfKeys` / `NumberOfSampledFrames` / `NumberOfSampledKeys`: int32
- `SamplingFrameRate` / `TargetFrameRate`: `(Numerator=30,Denominator=1)`
- `Skeleton`: object path string
- `BoneCompressionSettings` / `CurveCompressionSettings` / `VariableFrameStrippingSettings`: object paths
- `AnimationTrackNames`: `["Bone_001","Bone_002","Bone_002_end"]`
- `AnimNotifyTracks`: `[{"TrackColor": "...", "TrackName": "1"}]` — track headers only, no events
- `RawDataGuid`, `DataModelInterface`, `AssetImportData` (FBX import metadata)

What is MISSING or invisible on the same asset:

- `Notifies` — not emitted at all (array empty on this clip, but the key is
  also absent, so a consumer can't tell "no notifies" from "tool dropped them").
- `AuthoredSyncMarkers` — same: absent.
- `AdditiveAnimType` — absent (defaults elided by the property walker).
- No derived `lengthSeconds` field (have to divide NumFrames by FrameRate).

On clips that DO have notifies/curves (e.g.
`Game/SoStylized/Demo/Pawn/Mannequin/Animations/ThirdPersonRun/properties.json`),
`Notifies` and `RawCurveData` *do* appear in `properties.json`, but they're
deep struct soup interleaved with dozens of unrelated property entries. The
recent fix from `B-asset-dump-rawcurvedata-text-blob` (`#3-verify-fix DONE`)
collapsed RawCurveData into a structured summary inside `properties.json`,
so curve key counts are reachable — but there is still no single concise
file an LLM can read to answer "what does this animation do on the timeline".

## Compared to peer assets

- `AnimMontage` — emits `bpir.txt` (montage graph) + has notifies inline in
  `properties.json`. Closest precedent. No `anim_montage.json` sidecar yet.
- `LevelSequence` — emits `level_sequence.json` (bindings/tracks/sections/keys)
  via `LevelSequenceDumpBuilder`, dispatched from
  `AssetDumpHandler::BuildAllFilesForAsset` and registered in `DumpFileNames`
  + `Canonical[]`. Pattern set by `F-asset-dump-level-sequence-summary`.
- `SoundCue` — emits a graph sidecar similarly.

## Related existing ticket (not silently merged)

`B-asset-dump-anim-skips-root-and-slot-bindings` (DONE) is widget-animation
specific — different system entirely (UMG `UWidgetAnimation` v1 bindings).
Does not cover `UAnimSequence` timeline data.

`B-asset-dump-rawcurvedata-text-blob` (DONE) is narrower: it only fixed the
single-line ExportText blob inside `properties.json` for `RawCurveData`.
It did not introduce a unified AnimSequence sidecar, did not surface
notifies / sync markers / additive type, and did not derive `lengthSeconds`
or restructure the data around an LLM-facing schema.

This ticket is filed as new on that basis.

**Fix:** Emit an `anim_sequence.json` sidecar with the LLM-relevant
projection in a single file:

```json
{
  "lengthSeconds": 33.30,
  "frameRate": { "numerator": 30, "denominator": 1 },
  "numFrames": 1000,
  "additiveType": "AAT_None",
  "skeletonAssetPath": "/App/Meshes/Clutch/SK_Clutch02_Skeleton",
  "rawTrackCount": 3,
  "notifies": [
    { "name": "...", "displayName": "...", "time": 0.0, "duration": 0.0, "branchingPoint": false }
  ],
  "curves": [
    { "name": "...", "type": "Float", "keyCount": 12 }
  ],
  "syncMarkers": [ { "name": "...", "time": 0.0 } ]
}
```

Walk `UAnimSequence::Notifies`, `RawCurveData` (FloatCurves /
TransformCurves / VectorCurves), `AuthoredSyncMarkers`, and derive
`lengthSeconds` from `SamplingFrameRate * NumFrames` (or read the
DataModel's `PlayLength`). Pattern matches existing type-specific
sidecars: add an `AnimSequenceDumpBuilder` under `Private/Handlers/Asset/`,
dispatch from `BuildAllFilesForAsset`, register
`DumpFileNames::AnimSequence` and add to `Canonical[]`.

## History
- `#1-initial-repro` `OPEN` reporter — 854 `UAnimSequence` dumps emit only `meta.json` + `properties.json` (no type-specific sidecar). Example: `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Meshes/Clutch/SK_Clutch02_Anim/` — meta.json + ~5 KB properties.json only. `properties.json` exposes `SequenceLength=33.299999237060547`, `NumFrames=1000`, `SamplingFrameRate=(Numerator=30,Denominator=1)`, `Skeleton`, `AnimationTrackNames=["Bone_001","Bone_002","Bone_002_end"]`, `AnimNotifyTracks` (track headers only), but `Notifies`, `AuthoredSyncMarkers`, `AdditiveAnimType` keys are entirely absent from output (defaults elided — consumer can't distinguish empty-set from dropped). `AssetDumpHandler::BuildAllFilesForAsset` has no `UAnimSequence` branch; falls through to generic property walk. Related: `B-asset-dump-rawcurvedata-text-blob` (DONE) collapsed only the RawCurveData ExportText blob; no unified timeline sidecar. `B-asset-dump-anim-skips-root-and-slot-bindings` (DONE) is UMG widget-animation, not UAnimSequence. Proposed: emit `anim_sequence.json` sidecar with `{lengthSeconds, frameRate, numFrames, additiveType, skeletonAssetPath, rawTrackCount, notifies, curves, syncMarkers}` via a new `AnimSequenceDumpBuilder`, matching the `LevelSequenceDumpBuilder` pattern from `F-asset-dump-level-sequence-summary`.
- `#2-add-animsequence-sidecar` `IN-REVIEW` developer — Added `AnimSequenceDumpBuilder.{h,cpp}` mirroring `LevelSequenceDumpBuilder`. Emits `anim_sequence.json` (assetKind/path/lengthSeconds/frameRate/numFrames/additiveType/skeletonAssetPath/rawTrackCount/notifies/curves/syncMarkers) via `UAnimSequenceBase::Notifies`, `UAnimSequence::AuthoredSyncMarkers` (`WITH_EDITORONLY_DATA`), `RawCurveData::FloatCurves`+`TransformCurves`, `GetPlayLength`/`GetSamplingFrameRate`/`GetNumberOfSampledFrames`, `GetDataModel()->GetNumberOfTransformTracks()` (`WITH_EDITOR`). Arrays sorted (notifies by time/name, curves by name, syncMarkers by time) for diff stability. Wired dispatch in `AssetDumpHandler.cpp::BuildAllFilesForAsset` after `ULevelSequence`; added `DumpFileNames::AnimSequence` and registered in `Canonical[]`. Regression test `TestAnimSequenceDumpBuilder.cpp` with `FAnimSequenceDumpBuilderShapeTest` and `FAnimSequenceAssetDumpWritesAnimSequenceAspectFileTest`. Counterfactual: reverting the dispatch branch makes the end-to-end test fail because `HasDumpFile(Result.WrittenPaths, DumpFileNames::AnimSequence)` returns false.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/Meshes/Clutch/SK_Clutch02_Anim` (empty-notify clip) writes `anim_sequence.json` with all spec fields (`lengthSeconds=33.299...`, `frameRate={numerator:30,denominator:1}`, `numFrames=1000`, `additiveType=AAT_None`, `skeletonAssetPath=/App/Meshes/Clutch/SK_Clutch02_Skeleton.SK_Clutch02_Skeleton`, `rawTrackCount=3`, empty `notifies`/`curves`/`syncMarkers` arrays — distinguishing empty-set from dropped). Cross-checked `/Game/SoStylized/Demo/Pawn/Mannequin/Animations/ThirdPersonRun`: `notifies` contains two `FootstepRun` entries with `time` (0.189..., 0.515...) and `branchingPoint=false`; `curves` contains `blendOrient1`/`blendParent1` (`type=Float`, `keyCount=2`); `rawTrackCount=68`. Both files appear in the file list returned by `asset.dump`.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
