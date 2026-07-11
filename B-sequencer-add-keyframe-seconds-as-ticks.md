---
id: B-sequencer-add-keyframe-seconds-as-ticks
title: "sequencer.add_keyframe writes the key at a DisplayRate-frame tick instead of a TickResolution tick — the keyframe lands ~1000x too early in time, success reported"
status: IN-REVIEW
severity: High
category: bug
tags: [sequencer, add_keyframe, units, display-frames-as-ticks, bug]
encounters: 1
lastSeen: 2026-07-11T08:02:35Z
claimedBy: fuzz2
claimedAt: 2026-07-11T11:25:02.9473864+03:00
---

# sequencer.add_keyframe writes the key at a DisplayRate-frame tick, not a TickResolution tick — the keyframe lands ~1000x too early

`sequencer.add_keyframe` (the modern seconds-based verb, distinct from the legacy
frame-numbered `sequence.add_keyframe`) takes a keyframe `time` in **seconds**
(param doc: "Time in seconds for the keyframe", `SequencerHandler.cpp:41`) and writes
a cubic key onto a float channel. It converts the seconds to **display-rate** frames
via `FFrameRate::AsFrameTime()` and hands that frame number straight to
`FMovieSceneFloatChannel::AddCubicKey()`, whose key times are indexed in the
MovieScene's **TickResolution**, not its DisplayRate. At the default 24 fps DisplayRate
/ 24000 TickResolution the two differ by 1000x, so a key requested at 5 s is written at
tick 120 (0.005 s) instead of tick 120000 (5 s) — **~1000x too early** — while the RPC
still reports success and echoes the requested seconds, so the caller trusts a lie.

This is the same `seconds -> DisplayRate frames -> tick-resolution store` root cause as
`B-sequencer-section-range-display-frames-as-ticks` (the three section-authoring
`SetRange` verbs), but here the wrong-unit frame is fed to a channel-key writer
(`AddCubicKey`) rather than a section `SetRange`, so it is tracked as its own ticket
(mirroring how `sequencer.set_properties`'s units drift is its own ticket
`E-sequencer-property-unit-drift`).

## Guilty source (verbatim)

`sequencer.add_keyframe` (SequencerHandler.cpp:135-139):

```cpp
FFrameRate DisplayRate = MovieScene->GetDisplayRate();
FFrameTime FrameTime = DisplayRate.AsFrameTime(TimeSeconds);
FFrameNumber FrameNumber = FrameTime.GetFrame();
FMovieSceneFloatChannel& Channel = FloatSection->GetChannel();
Channel.AddCubicKey(FrameNumber, static_cast<float>(Value));
```

`AddCubicKey` takes a key time in `MovieScene->GetTickResolution()`. Passing a
DisplayRate frame number is off by the `TickResolution / DisplayRate` ratio
(24000/24 = 1000 at defaults).

## What it should do

Convert seconds to **tick-resolution** frames before `AddCubicKey`, e.g.
`MovieScene->GetTickResolution().AsFrameNumber(TimeSeconds)` — the same conversion the
sibling ticket applies to the three `SetRange` section-authoring verbs.

## Acceptance

- Add a float track + key via `sequencer.add_keyframe` at `time=5` on a sequence at
  default 24000 TickResolution.
- The written key's frame equals `TickResolution.AsFrameNumber(5.0)` (120000 at
  defaults), NOT the DisplayRate-frame value (120 at 24 fps / 150 at 30 fps).
- Converting the stored key frame back to seconds recovers ~5 s.

## History
- `#1-split-from-section-range-ticket` `OPEN` reporter — Split from
  `B-sequencer-section-range-display-frames-as-ticks` during that ticket's fix. The
  three `SetRange` section-authoring verbs were fixed there, but independent source
  review confirmed `sequencer.add_keyframe` (SequencerHandler.cpp:135-139) shares the
  identical `seconds -> DisplayRate frames -> tick store` root cause on a different
  sink (`FMovieSceneFloatChannel::AddCubicKey`, not a section `SetRange`). Kept as its
  own ticket rather than folded in, per the one-family-per-ticket convention (cf.
  `E-sequencer-property-unit-drift`). Distinct from the legacy frame-numbered
  `sequence.add_keyframe` (`SequenceHandler.cpp`, which converts correctly via
  `SequenceHelpers::DisplayFrameToTick`) and from the response-shape / value-shape /
  section-expansion keyframe tickets (different root causes).
- `#2-fix-in-review` `IN-REVIEW` developer — GO: confirmed the defect verbatim in
  current source (`Handlers/Sequencer/SequencerHandler.cpp:150-154` converts seconds via
  `DisplayRate.AsFrameTime(...).GetFrame()` and hands the DisplayRate frame straight to
  `FMovieSceneFloatChannel::AddCubicKey`, ~1000x too early at defaults / ~800x at this
  host's 30fps DisplayRate). Root-cause fix: convert seconds to tick-resolution frames via
  the co-located `SequencerSectionHelpers::SecondsToTickFrame` helper (the same one the three
  sibling `SetRange` verbs already use), keeping the conversion in one place. Adopting the
  red test as the regression test.
