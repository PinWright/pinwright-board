---
id: F-sequencer-camera-cut-binding-readback
title: "No reader surfaces a camera-cut section's target camera (CameraBindingID) — get_camera_cut_track / list_sections / asset.dump all omit it"
status: IN-REVIEW
severity: Medium
category: feature
tags: [sequencer, readback, parity, camera-cut, round-trip]
encounters: 1
lastSeen: 2026-07-13T10:15:34.7114018+03:00
---

# No reader surfaces a camera-cut section's target camera (`CameraBindingID`)

`sequencer.add_camera_track` records **which camera a cut points at** as the
section's `CameraBindingID` — `CameraCutSection->SetCameraBindingID(FMovieSceneObjectBindingID(CameraGuid))`
(`SequencerHandler.cpp:408`). That binding GUID is the single defining field of a
camera-cut section: it is what makes the sequence play back through a particular
camera. But **no sequencer reader surfaces it.** A caller who authored a
camera-cut track cannot confirm the cut references the intended camera through
any dedicated sequencer read surface — exactly the "reopen the sequence and
confirm the cut points at the right camera" check a cinematic task ends on.

## Root cause (shared helper)

Every reader that emits a camera-cut section's JSON goes through the shared
`MovieSceneJsonUtils::BuildSectionJson` (`Plugins/PinWright/Source/PinWright/Private/Utils/MovieSceneJsonUtils.h:213-252`),
which emits only `range`, `blendType`, `rowIndex`, `isLocked`, `channels`
(and, for sub-sections, `innerSequencePath`/`timeScale`). It has **no
`UMovieSceneCameraCutSection` branch**, so the section's `CameraBindingID` is
never read back. A camera-cut section carries no scalar `channels` either
(`channels: []`), so the target camera falls out of the payload entirely.

## Affected readers (all share `BuildSectionJson`)

- **`sequencer.get_camera_cut_track`** — `SequenceHandler.cpp:3261` → `BuildTrackJson` → `BuildSectionJson`. Reproduced live (below).
- **`sequencer.list_sections`** — `SequenceHandler.cpp:3344` → `SequenceHelpers::BuildListedSectionJson` → `BuildSectionJson`. Reproduced live (below).
- **`asset.dump` / `level_sequence.json`** — `LevelSequenceDumpBuilder.cpp:93` → the same `BuildTrackJson`/`BuildSectionJson`. Enumerated via the shared-helper sweep (same code path; the DONE ticket `F-rpc-sequencer-get-camera-cut-track` already established `get_camera_cut_track` sections are byte-identical to `level_sequence.json#cameraCutTrack`), not independently re-run this iteration.

## Misleading adjacent field (secondary)

`sequencer.list_sections` *does* emit a `bindingGuid` on the camera-cut section,
but it is `""` — that field is the master **track's** object binding (genuinely
empty for the camera-cut master track), NOT the section's target camera. A caller
could misread `bindingGuid: ""` as "this cut targets no camera" when the section
actually references a valid camera binding that just isn't surfaced.

## What it should do

`BuildSectionJson` should special-case `UMovieSceneCameraCutSection` and emit the
resolved target binding, e.g. `cameraBindingId: Section->GetCameraBindingID().GetGuid().ToString()`
(guard the invalid/empty GUID as JSON `null`), so `get_camera_cut_track`,
`list_sections`, and `asset.dump` all round-trip the camera a cut points at.
Then a caller can assert the cut references the intended camera binding GUID
directly instead of inferring it from a one-binding invariant.

## Verbatim repro (replay-confirmed live at HEAD)

Asset: `/Game/Cinematics/IntroShowcase` (one bound camera "SequenceCamera",
binding GUID `5EF2B6C64E267C70AC88E5A2A0624B97`; camera-cut section over
ticks 0..120000).

1. `sequencer.get_bindings {path:/Game/Cinematics/IntroShowcase}` ->
   `{"bindings":[{"id":"5EF2B6C64E267C70AC88E5A2A0624B97","name":"SequenceCamera"}]}`
2. `sequencer.get_camera_cut_track {path:/Game/Cinematics/IntroShowcase}` ->
   `{"cameraCutTrack":{"name":"None","class":"MovieSceneCameraCutTrack","isEvalDisabled":false,"sectionCount":1,"sections":[{"range":{"start":0,"end":120000},"blendType":"Absolute","rowIndex":0,"isLocked":false,"channels":[]}]}}`
   — no `cameraBindingId`; the GUID `5EF2B6C64E267C70AC88E5A2A0624B97` is absent.
3. `sequencer.list_sections {path:/Game/Cinematics/IntroShowcase}` -> the camera-cut row is
   `{"range":{"start":0,"end":120000},"blendType":"Absolute","rowIndex":0,"isLocked":false,"channels":[],"trackName":"None","trackClass":"MovieSceneCameraCutTrack","bindingGuid":""}`
   — `bindingGuid:""` is the empty track binding, not the cut's target; the section's real `CameraBindingID` (`5EF2...`) is nowhere.

## Guilty source (verbatim)

Write side — the binding IS stored (`SequencerHandler.cpp:406-408`):

```cpp
if (CameraGuid.IsValid())
{
    CameraCutSection->SetCameraBindingID(FMovieSceneObjectBindingID(CameraGuid));
}
```

Read side — the shared section serializer never reads it back
(`Utils/MovieSceneJsonUtils.h:213-225`):

```cpp
inline TSharedPtr<FJsonObject> BuildSectionJson(const UMovieSceneSection* Section, bool bIncludeKeys = false)
{
    TSharedPtr<FJsonObject> Obj = MakeShared<FJsonObject>();
    if (!Section) { return Obj; }
    Obj->SetObjectField(TEXT("range"), MakeFrameRangeObject(Section->GetRange()));
    Obj->SetStringField(TEXT("blendType"), BlendTypeToString(Section->GetBlendType()));
    Obj->SetNumberField(TEXT("rowIndex"), Section->GetRowIndex());
    Obj->SetBoolField(TEXT("isLocked"), Section->IsLocked());
    Obj->SetArrayField(TEXT("channels"), BuildChannelEntriesJson(Section->GetChannelProxy(), bIncludeKeys));
    // ... sub-section fields only; no UMovieSceneCameraCutSection / CameraBindingID branch ...
}
```

severity rationale: impact=readback omits a field (the cut's target camera is unverifiable through the dedicated sequencer read surface; only a `python.execute` drop-out to `Section.get_camera_binding_id()` recovers it) × reach=rare (camera-cut authoring, a normal but not every-session cinematic path) -> Medium. Capped at Medium regardless: a working `python.execute` path exists, and the omission is honest-but-incomplete (not a false value the caller trusts).

## See also
- `F-rpc-sequencer-get-camera-cut-track` (DONE) — added the `get_camera_cut_track` reader itself (track visibility parity); its sections never included `CameraBindingID`, so this is the completeness follow-up, not a regression of it.
- `F-sequencer-track-state-readback` (IN-REVIEW) — sibling readback-parity gap (mute/solo/lock) that recently extended `BuildSectionJson`/`BuildTrackJson` with `isLocked`/`isEvalDisabled` yet still did not add the camera binding.

## History
- `#2-readback-branch` `IN-REVIEW` developer — Added a `UMovieSceneCameraCutSection` branch to the shared section serializer `MovieSceneJsonUtils::BuildSectionJson` (`Plugins/PinWright/Source/PinWright/Private/Utils/MovieSceneJsonUtils.h`): it now emits `cameraBindingId = GetCameraBindingID().GetGuid().ToString()` (default = Digits, matching `get_bindings`' `id` format so a caller can compare directly), guarding an unset/invalid GUID as JSON `null` rather than a misleading `""`. Because all three readers funnel through this one helper, the fix lands `cameraBindingId` on `get_camera_cut_track`, `list_sections`, and `asset.dump`/`level_sequence.json` at once; added the `Sections/MovieSceneCameraCutSection.h` include. Bumped the `level_sequence.json` aspect version `3`→`4` in `AssetDumpCache.cpp` (serialized bytes changed) so stale dump caches regenerate. Regression test `PinWright.Sequencer.CameraCutBindingReadback.SectionSurfacesCameraBindingId` (`Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestCameraCutBindingReadback.cpp`) builds two in-code transient camera-cut sections and asserts BuildSectionJson round-trips a stamped binding GUID and emits JSON `null` for an unbound cut; reverting the branch fails it. Files: `Utils/MovieSceneJsonUtils.h`, `Handlers/Asset/AssetDumpCache.cpp`, `Tests/Sequencer/TestCameraCutBindingReadback.cpp`.
- `#1-initial-repro` `OPEN` reporter — REALISM-mode establishing-shot cinematic task (`sequencer.create` .. `add_camera_track` .. save; 16 touched methods). Root cause replay-confirmed live at HEAD on `/Game/Cinematics/IntroShowcase`: `get_camera_cut_track` and `list_sections` both return the camera-cut section as `{range,blendType,rowIndex,isLocked,channels:[]}` with no `cameraBindingId`, while `get_bindings` shows the one camera binding is `5EF2B6C64E267C70AC88E5A2A0624B97` (the GUID `add_camera_track` fed to `SetCameraBindingID`). Root: shared `MovieSceneJsonUtils::BuildSectionJson` (`MovieSceneJsonUtils.h:213-252`) has no `UMovieSceneCameraCutSection` branch, so all three readers that go through `BuildTrackJson`/`BuildListedSectionJson` (`sequencer.get_camera_cut_track` `SequenceHandler.cpp:3261`, `sequencer.list_sections` `:3344`, `asset.dump` `LevelSequenceDumpBuilder.cpp:93`) omit the cut's target camera. Dedup: ripgrep OPEN/DONE/WONTFIX for camera-cut / CameraBindingID / camera binding / target camera — `F-rpc-sequencer-get-camera-cut-track` (DONE, added the reader), `E-sequencer-list-tracks-omits-camera-cut-undocumented` (list_tracks omits the whole track), `B-sequencer-remove-track-misses-camera-cut-slot` (mutator can't reach the slot), `F-sequencer-track-state-readback` (mute/solo/lock fields) — none cover the section's `CameraBindingID` omission. Genuinely new; family-level across the three readers that share `BuildSectionJson`.
