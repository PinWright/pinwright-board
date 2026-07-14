---
id: B-sequencer-section-range-display-frames-as-ticks
title: "sequencer section-authoring verbs (add_camera_track / add_animation_track / add_audio_track) size sections with DisplayRate frames fed into a TickResolution SetRange — sections come out ~1000x too short, success reported"
status: IN-REVIEW
severity: High
category: bug
tags: [section-range-display-frames-as-ticks, sequencer, units, add_camera_track, add_animation_track, add_audio_track, bug]
encounters: 1
lastSeen: 2026-07-11T09:58:44.1621854+03:00
---

# sequencer section-authoring verbs size sections in display-rate frames but SetRange stores tick-resolution frames — sections are ~1000x too short

Three sibling sequencer verbs that take a duration in **seconds** and author a
`UMovieSceneSection` share one broken conversion: they turn the seconds into
**display-rate** frames via `FFrameRate::AsFrameTime()` and then hand those frame
numbers straight to `UMovieSceneSection::SetRange()`, which stores frame numbers in
the MovieScene's **TickResolution**, not its DisplayRate. At the default 24 fps
DisplayRate / 24000 TickResolution the two differ by a factor of 1000, so every
authored section spans `seconds * DisplayRate` **ticks** instead of
`seconds * TickResolution` ticks — a **~1000x-too-short** section. The RPC still
returns `success: true` and echoes the caller's requested seconds, so the caller
trusts a lie.

Affected methods (all in `Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequencerHandler.cpp`):

- `sequencer.add_camera_track` — the camera-cut section
- `sequencer.add_animation_track` — the skeletal-animation section
- `sequencer.add_audio_track` — the audio section

For `add_camera_track` this means "cut to that camera for the whole shot" produces
a camera-cut section that covers only the first ~0.005 s of a 5 s shot — the cut
track exists and passes a shallow "does a cut track exist" check, but does not span
the shot.

## Guilty source (verbatim)

`add_camera_track` (SequencerHandler.cpp:303-313):

```cpp
FFrameRate DisplayRate = MovieScene->GetDisplayRate();
FFrameTime StartFrameTime = DisplayRate.AsFrameTime(StartTime);
FFrameTime EndFrameTime = DisplayRate.AsFrameTime(EndTime);
FFrameNumber StartFrame = StartFrameTime.GetFrame();
FFrameNumber EndFrame = EndFrameTime.GetFrame();
...
CameraCutSection->SetRange(TRange<FFrameNumber>(StartFrame, EndFrame));
```

`add_animation_track` (SequencerHandler.cpp:649-654):

```cpp
FFrameRate DisplayRate = MovieScene->GetDisplayRate();
FFrameTime StartFrame = DisplayRate.AsFrameTime(StartTime);
float AnimLength = AnimSequence->GetPlayLength();
FFrameTime EndFrame = DisplayRate.AsFrameTime(StartTime + AnimLength);
AnimSection->SetRange(TRange<FFrameNumber>(StartFrame.GetFrame(), EndFrame.GetFrame()));
```

`add_audio_track` (SequencerHandler.cpp:810-813):

```cpp
const FFrameRate DisplayRate = MovieScene->GetDisplayRate();
const FFrameNumber StartFrame = DisplayRate.AsFrameTime(StartTime).GetFrame();
const FFrameNumber EndFrame = DisplayRate.AsFrameTime(StartTime + Duration).GetFrame();
Section->SetRange(TRange<FFrameNumber>(StartFrame, EndFrame));
```

`SetRange` on a `UMovieSceneSection` stores frame numbers in `MovieScene->GetTickResolution()`.
Passing DisplayRate frame numbers is off by the `TickResolution / DisplayRate` ratio
(24000/24 = 1000 at defaults).

## What it should do

Convert seconds to **tick-resolution** frames before `SetRange`, e.g.
`MovieScene->GetTickResolution().AsFrameNumber(TimeSeconds)`, or convert the display
`FFrameTime` to ticks via `FFrameRate::TransformTime(DisplayRate.AsFrameTime(t), DisplayRate, TickResolution)`.
The correct pattern already exists nearby: the transform section is ranged from
`MovieScene->GetPlaybackRange()` (ticks) at SequencerHandler.cpp:421, and
`sequencer.set_properties` correctly converts its display-frame playback range to
ticks (live replay: `playbackEnd:120` -> stored `120000` ticks). These three
section-authoring verbs are the odd ones out.

Excluded from this family: `sequencer.add_level_visibility_track`
(SequencerHandler.cpp:541) takes raw frame-number input, not seconds, so it uses a
different (frames-as-ticks) contract and is not part of this DisplayRate.AsFrameTime
family. Distinct from `E-sequencer-property-unit-drift` (a different method,
`set_properties`, in a different file `SequenceHandler.cpp`, whose bug is a missing
display->tick conversion on frame-number input — related symptom, different root
cause and code path).

## Verbatim repro (live replay this finding, HEAD)

Tick resolution 24000, display rate 24 fps (`sequencer.get_properties` readback:
`tickResolution {numerator:24000}`, `frameRate {numerator:24}`, `playbackEnd:120000`).

- `sequencer.create` `{name:"ReplayCutRange", path:"/Game/Cinematics"}` -> saved.
- `sequencer.set_properties` `{path:"/Game/Cinematics/ReplayCutRange", frameRate:24, playbackStart:0, playbackEnd:120}` -> `playbackEnd:120000` ticks (5 s), `applied:true`.
- `sequencer.add_camera` `{path:"/Game/Cinematics/ReplayCutRange"}` -> camera `CameraActor_7`.
- `sequencer.add_camera_track` `{sequencePath:"/Game/Cinematics/ReplayCutRange", cameraActorPath:".../CameraActor_7", startTime:0, endTime:5}` -> `{success:true, startTime:0, endTime:5}`.
- `sequencer.list_sections` `{path:"/Game/Cinematics/ReplayCutRange"}` ->
  `sections:[{range:{start:0, end:120}, trackClass:"MovieSceneCameraCutTrack", ...}]`.

Expected cut section range for a 5 s shot at TickResolution 24000: `{start:0, end:120000}`.
Actual: `{start:0, end:120}` = 120 ticks = 0.005 s — 1000x too short. The playback range
(120000 ticks) and the transform section (authored from GetPlaybackRange) are correct
by contrast, so the defect is confined to the seconds->SetRange conversion in these
three verbs.

severity rationale: impact=silent-wrong-data on a normal path (section authored
~1000x too short while `success:true` echoes the requested seconds; no in-verb
workaround) x reach=sequencer section-authoring family (camera-cut / skeletal-anim /
audio — normal cinematic path) -> High.

## History
- `#1-initial-repro` `OPEN` reporter — Live-replayed `add_camera_track` at HEAD
  (create -> set_properties 24fps -> add_camera -> add_camera_track 0..5s ->
  list_sections): cut section came out `{start:0,end:120}` ticks (0.005 s) instead
  of `{start:0,end:120000}` (5 s). Source read confirmed the shared root cause
  (`DisplayRate.AsFrameTime(seconds).GetFrame()` fed into a TickResolution
  `SetRange`) is identical across `add_camera_track` (SequencerHandler.cpp:303-313),
  `add_animation_track` (:649-654), and `add_audio_track` (:810-813) — filed as one
  family ticket. Dedup: ripgrep OPEN/IN-REVIEW/DONE/WONTFIX — no existing ticket
  covers these seconds-input section-range verbs;
  `E-sequencer-property-unit-drift` (IN-REVIEW) is a distinct method/file/contract
  (`set_properties` frame-number input, `SequenceHandler.cpp`), and
  `F-sequencer-transform-section-range-not-expanded-by-keyframe` is a
  collapsed-range keyframe issue, not a misconversion. Surfaced by a
  `sequencer.create` seed task (5 s IntroFlyby cinematic) whose doer flagged the
  cut-section length as a hunch; confirmed here by replay + source.
- `#2-go-fix-three-setrange-verbs` `IN-REVIEW` developer — Disposition GO. Root
  cause independently re-confirmed at all three cited sites in source
  (`add_camera_track` :303-313, `add_animation_track` :649-654, `add_audio_track`
  :810-813): seconds -> DisplayRate frames via `AsFrameTime` fed to a TickResolution
  `SetRange`. Fixing by converting seconds -> TICK frames
  (`GetTickResolution().AsFrameNumber(seconds)`) at all three. Scope held to these
  three section-authoring SetRange verbs. The identical seconds-as-ticks root cause
  in the channel-key writer `sequencer.add_keyframe` (SequencerHandler.cpp:135-139,
  `AddCubicKey` — a different sink, not a section `SetRange`) is NOT folded in; it is
  split to its own new ticket `B-sequencer-add-keyframe-seconds-as-ticks` (mirroring
  the set_properties-as-own-ticket precedent `E-sequencer-property-unit-drift`).
  Adopting the red test
  `PinWright.Sequencer.SectionRange.CameraCutSpansRequestedSeconds` as the regression
  gate.
- `#3-shipped-verified-green` `IN-REVIEW` developer — Shipped: converted seconds ->
  TICK frames via `GetTickResolution().AsFrameNumber()` at all three sites in
  `Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequencerHandler.cpp`
  (add_camera_track :304-313, add_animation_track :650-655, add_audio_track :811-814).
  Plugin compiled clean (EAContentExamples57Editor, Result: Succeeded). Adopted red
  test `PinWright.Sequencer.SectionRange.CameraCutSpansRequestedSeconds`
  (`Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSequencerSectionRangeUnits.cpp`)
  flipped red -> green: the camera-cut section now stores `[0, 120000)` ticks =
  5.000000 s (pre-fix stored end=150, ~1000x too short). add_keyframe sibling filed
  OPEN as `B-sequencer-add-keyframe-seconds-as-ticks`.
