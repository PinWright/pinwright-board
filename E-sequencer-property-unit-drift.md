---
id: E-sequencer-property-unit-drift
title: "sequencer.set_properties stores frame-labeled playbackStart/playbackEnd as raw ticks (silent ~1000x-wrong playback range)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [sequencer, units, set_properties, playback-range, bug]
encounters: 3
lastSeen: 2026-07-02T13:30:46.7600928+03:00
---

# sequencer.set_properties stores frame-labeled playbackStart/playbackEnd as raw ticks (silent ~1000x-wrong playback range)

`sequencer.set_properties` advertises `playbackStart` / `playbackEnd` /
`lengthInFrames` as **display-rate frame numbers** — the param schema literally
labels them `"Playback start frame"` / `"Playback end frame"` /
`"Length in frames from start"` (`SequenceHandler.cpp:435-437`). But the handler
casts the value straight into a **tick-resolution** `FFrameNumber` with **no**
display→tick conversion:

```cpp
if (bHasPlaybackStart) StartFrame = FFrameNumber(static_cast<int32>(PlaybackStartValue)); // :497
if (bHasPlaybackEnd)   EndFrame   = FFrameNumber(static_cast<int32>(PlaybackEndValue));   // :499
MovieScene->SetPlaybackRange(TRange<FFrameNumber>(StartFrame, EndFrame));                 // :505
```

`SetPlaybackRange` stores tick-resolution frame numbers, so a value labeled a
"frame" is silently interpreted as raw ticks. At 24 fps / `tickResolution 24000`,
`set_properties {playbackEnd:120}` (intended = frame 120 = 5 s) stores **120 ticks
= 0.005 s** — a **~1000x error** — and the response still returns `applied:true`
with no warning. Getting a real 5 s range today requires passing the raw
tick value `playbackEnd:120000`, which contradicts the documented "frame"
contract. This is the concrete, source-confirmed, live-replayed defect (encounter
`#2`), and it is inconsistent with the keyframe path, which *does* convert via
`SequenceHelpers::DisplayFrameToTick` (`SequenceHandler.cpp:225-231`, called at
`:1652`/`:1761`).

**The original "unit drift across readers/writers" framing was a misdiagnosis**
(corrected against source; see history for the encounter narratives that led there):

- `sequencer.get_properties` does **not** flip units after a write. It
  unconditionally returns raw ticks (`Range.Get{Lower,Upper}BoundValue().Value`,
  `SequenceHandler.cpp:1206-1211`) with no branch on write history. Encounter
  `#1`'s "48000 ticks before, 150 frames after" was the `#2` write bug storing
  `150` raw ticks (mis-cast), read back as `150` ticks — the range *shrank*, it
  did not switch to display-frames.
- `sequencer.list_sections` does **not** emit two units in one payload.
  `MovieSceneJsonUtils::MakeFrameRangeObject` and `BuildChannelKeysJson`
  (`Utils/MovieSceneJsonUtils.h:39,43,91,130`) serialize **every** section range
  and key as raw `FFrameNumber.Value`, with no per-track-type unit branching.
  Encounter `#3`'s `end:120` (camera-cut) vs `end:120000` (transform) in one
  payload are two genuinely different **tick** ranges — the camera-cut range
  authored via the un-converted `set_properties`/camera path (the `#2` bug), the
  transform keys via the converting keyframe path — not two units from one
  serializer.

Secondary (real but not the core defect): `set_view_range` takes **seconds**
(`SequenceHandler.cpp:2097-2098,2117`) while `set_properties` takes frames — an
ergonomic frames-vs-seconds split, an aside to the write bug.

**Fix (implemented):** convert display frames → ticks in the `set_properties`
handler via the existing `SequenceHelpers::DisplayFrameToTick` for
`playbackStart` / `playbackEnd` / `lengthInFrames`, matching the keyframe path,
so the stored range honors the documented "frame" contract. The response keeps
echoing tick-resolution values (consistent with `get_properties`) and now also
publishes `tickResolution` so the echoed ticks are self-describing. Param
descriptions are sharpened to name the unit ("display-rate frame number").

**Note — behavioral/breaking:** callers who worked around this by passing raw
ticks (`playbackEnd:120000`) will now double-scale; the params were always
documented as frames, so this restores the documented contract rather than
inventing a new one.

## History
- `#1-initial-audit` `OPEN` reporter — Audit-then-standardize pass on `2_1_TransformTracks_LS` (12 calls, all ok=true, outcome nonrepro). Friction note: "get_properties reports playbackEnd in tick-resolution units before set_properties (48000 ticks @24000 = 2s) yet in display-frame units after (150 frames), and set_properties takes playbackStart/playbackEnd in frames while set_view_range takes seconds — the mixed/shifting units (ticks vs display-frames vs seconds across sibling readers/writers) forced me to do conversions from tickResolution and could easily mislead a user comparing the two get_properties reads." Distinct from `E-rpc-sequencer-extend-get-properties` (DONE), which *added* tickResolution/count fields but did not address the unit drift in the existing playback fields nor the frames-vs-seconds split between `set_properties` and `set_view_range`. Downstream wiki target: `docs/wiki-src/sequencer.md` (get_properties / set_properties / set_view_range H3s).
- `#3-additional-list-sections-mixed-units-in-one-payload` `OPEN` reporter — New angle extending the family to the `sequencer.list_sections` reader (struggle-audit PROCESS of the `CS_Establishing` 24fps 0-5s establishing-shot authoring task, focus `sequencer`, 31 calls, outcome ergo). A SINGLE `sequencer.list_sections{includeKeys:true}` response reports two sections of the same 0-5s shot in TWO different units: the `MovieScene3DTransformTrack` section range + keyframes come back in **tick-resolution units** (`end:120000`, key at frame `120000`) while the `MovieSceneCameraCutTrack` section range comes back in **display frames** (`end:120`) — in the same payload. This forced the caller to spend narration reconciling that both spans are actually 5s (SAY: "the camera-cut section still reads {start:0, end:120} — hmm, its range is displayed in display frames (0-120) while the transform section reads in tick units (0-120000)... a units display difference"). A consumer verifying "keyframes at distinct frames" cannot compare a key at frame `120000` against a section ending at frame `120` without knowing which unit each is in. This is distinct from `#1`/`#2` (which are the get_properties/set_properties/set_view_range PLAYBACK fields) — here the drift is BETWEEN sibling sections WITHIN one `list_sections` payload, so `list_sections` is now added to the affected-methods/tags. Proposed fix: normalize all frame numbers in `list_sections` to one unit (prefer display frames, matching the Sequencer UI) OR emit an explicit per-value unit tag / a top-level `{frameRate,tickResolution}` so readers can convert. Downstream wiki page: `docs/wiki-src/sequencer.md` (`list_sections` H3). Same underlying frame-vs-tick root as `#1`/`#2`; no calls errored (all 31 ok=true).
- `#2-additional-set-properties-is-ticks-not-frames` `OPEN` reporter — Additional evidence (new angle, source-confirmed + live-replayed on `/Game/Cinematics/CS_ReplayUnits`): the original `#1` claim that `set_properties` takes `playbackStart`/`playbackEnd` "in frames" is WRONG — it takes **raw tick-resolution units**, while the param schema advertises them as display frames. `SequenceHandler.cpp:497,499` builds `StartFrame = FFrameNumber(static_cast<int32>(PlaybackStartValue))` / `EndFrame = FFrameNumber(static_cast<int32>(PlaybackEndValue))` — a bare cast into `FFrameNumber` with **no** `FFrameRate::TransformTime(displayRate → tickResolution)` conversion (contrast the keyframe path's `SequenceHelpers::DisplayFrameToTick`, `SequenceHandler.cpp:225-231`, which *does* convert). Yet `SequenceHandler.cpp:435-436` documents the params as `RPC_PARAM_OPT("playbackStart", "number", "Playback start frame")` / `"Playback end frame"`. Verbatim repro at 24 fps (tickResolution 24000/1): `sequencer.set_properties {path, playbackStart:0, playbackEnd:120}` → `{"playbackStart":0,"playbackEnd":120,"duration":120,"applied":true}` — a caller intending "frame 120 = 5 s" silently gets a **120-tick = 0.005 s** range (a ~1000× error) with `applied:true` and no warning. Getting a real 5 s range requires `playbackEnd:120000` (raw ticks), confirmed: `set_properties {playbackEnd:120000}` → `duration:120000`, and `get_properties` echoes `playbackStart:0, playbackEnd:120000, duration:120000, tickResolution:{24000,1}` — i.e. `get_properties` is **also ticks here**, consistent with `set_properties`, NOT the display-frame flip `#1` observed (so the readback unit may itself vary by path/asset — reinforcing the drift). The mislabeled `"frame"` param is the concrete quotable defect: fix is (a) rename/annotate the params as tick-resolution units OR actually convert display frames → ticks inside the handler (using `DisplayFrameToTick` like the keyframe path), and (b) annotate `get_properties`/`set_properties`/`set_view_range` units in `docs/wiki-src/sequencer.md`. Encountered in a fresh establishing-shot authoring task (build CS_Establishing 24fps 0-5s); outcome ergo, culprit `sequencer.set_properties`.
- `#4-reword-and-fix-set-properties-frame-tick-conversion` `IN-REVIEW` developer — REWORD + fix. Verified all three encounters against current source: the `#2` write bug is real (`SequenceHandler.cpp:435-437` params labeled "...frame" but `:497,499` cast straight into a tick-resolution `FFrameNumber` with no conversion, `SetPlaybackRange` at `:505`), while `#1` (get_properties "unit flip") and `#3` (list_sections "two units in one payload") are misdiagnoses — `get_properties` is unconditionally raw ticks (`:1206-1211`) and `MovieSceneJsonUtils.h:39,43,91,130` emits raw ticks uniformly with no per-track-type branching. Reworded title/body/severity(Low→Medium)/category(ergonomic→bug) to center the write-side defect and correct the reader-drift framing; dropped the docs-only "annotate the flip" fix (it would document behavior the code never exhibits). Fix: route `playbackStart`/`playbackEnd`/`lengthInFrames` through the existing `SequenceHelpers::DisplayFrameToTick` (the helper the keyframe path already uses), sharpen the param labels to "display-rate frame number", and publish `tickResolution` in the response so the echoed tick values are self-describing. Files: `Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`. Regression test added: `PinWright.Sequencer.SetProperties.PlaybackRangeFramesToTicks` (`Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestSetPropertiesFramesToTicks.cpp`) — drives the real registered handler on an in-code transient sequence at 24fps/24000-tick, asserts the stored playback range is the frame→tick conversion (30→30000, 120→120000) and NOT the raw unconverted frames; reverting the conversion makes it fail. (Correctness lens argued this is min Medium, arguably High as silent-wrong-data on the documented contract; set Medium as it is confined to one optional writer.)
