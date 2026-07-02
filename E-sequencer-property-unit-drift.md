---
id: E-sequencer-property-unit-drift
title: "sequencer playback fields shift units (ticks vs display-frames vs seconds) across sibling readers/writers"
status: OPEN
severity: Low
category: ergonomic
tags: [sequencer, units, docs, get_properties, set_properties, set_view_range, list_sections]
encounters: 3
lastSeen: 2026-07-02T13:30:46.7600928+03:00
---

# sequencer playback fields shift units (ticks vs display-frames vs seconds) across sibling readers/writers

When auditing then standardizing a level sequence, the playback-range time unit
is not stable across the sequencer namespace's own readers and writers, with no
unit annotation in the response or params:

- `sequencer.get_properties` reports `playbackEnd` in **tick-resolution units**
  on a freshly-opened asset (observed `48000` ticks at `tickResolution 24000`
  = 2 s), but in **display-frame units** after a `set_properties` write
  (observed `150` frames at 30 fps = 5 s). The same field name on the same
  method returns two different units depending on whether the asset's range was
  last written through the RPC. A caller who diffs an initial vs a confirmation
  `get_properties` read (the natural audit pattern) sees `48000 → 150` and can
  reasonably conclude the range *shrank*, when it actually grew from 2 s to 5 s.
- `sequencer.set_properties` takes `playbackStart` / `playbackEnd` in **frames**,
  while its sibling `sequencer.set_view_range` takes **seconds** — so framing the
  whole timeline after setting a 0–150-frame range means hand-converting 150
  frames back to `0–5 s` against the display rate to call `set_view_range`.

The values are all internally correct; the problem is purely ergonomic — the
unit is implicit and *shifts*, forcing the caller to carry `tickResolution` and
`displayRate` and convert between ticks, display-frames, and seconds to reason
about a single timeline. This is the same "mixed time units" hazard already
flagged for `add_keyframe` (seconds-based vs frame-numbered shapes) in the
sequencer wiki, surfacing here on the property reader/writer triple.

**What it should do:** either (a) make `get_properties.playbackStart/End`
report in one fixed, documented unit regardless of write history (display-frames
is the natural choice given `set_properties` writes frames), ideally echoing the
unit alongside the value (e.g. a `playbackRangeUnit: "displayFrames"` field or
parallel `*Seconds` fields), and/or (b) at minimum document the unit of every
playback field on each method. Since the numbers are already correct, the
cheapest first step is a **docs** fix: name and annotate the units on the
`sequencer.get_properties` / `sequencer.set_properties` / `sequencer.set_view_range`
H3 sections of `docs/wiki-src/sequencer.md`, explicitly warning that
`get_properties.playbackEnd` may come back in ticks on an untouched asset and in
display-frames after a write, and that `set_properties` is frames while
`set_view_range` is seconds.

**Workaround:** read `tickResolution` and `frameRate` from `get_properties`
first and convert all playback values to seconds yourself before comparing reads
or calling `set_view_range`.

## History
- `#1-initial-audit` `OPEN` reporter — Audit-then-standardize pass on `2_1_TransformTracks_LS` (12 calls, all ok=true, outcome nonrepro). Friction note: "get_properties reports playbackEnd in tick-resolution units before set_properties (48000 ticks @24000 = 2s) yet in display-frame units after (150 frames), and set_properties takes playbackStart/playbackEnd in frames while set_view_range takes seconds — the mixed/shifting units (ticks vs display-frames vs seconds across sibling readers/writers) forced me to do conversions from tickResolution and could easily mislead a user comparing the two get_properties reads." Distinct from `E-rpc-sequencer-extend-get-properties` (DONE), which *added* tickResolution/count fields but did not address the unit drift in the existing playback fields nor the frames-vs-seconds split between `set_properties` and `set_view_range`. Downstream wiki target: `docs/wiki-src/sequencer.md` (get_properties / set_properties / set_view_range H3s).
- `#3-additional-list-sections-mixed-units-in-one-payload` `OPEN` reporter — New angle extending the family to the `sequencer.list_sections` reader (struggle-audit PROCESS of the `CS_Establishing` 24fps 0-5s establishing-shot authoring task, focus `sequencer`, 31 calls, outcome ergo). A SINGLE `sequencer.list_sections{includeKeys:true}` response reports two sections of the same 0-5s shot in TWO different units: the `MovieScene3DTransformTrack` section range + keyframes come back in **tick-resolution units** (`end:120000`, key at frame `120000`) while the `MovieSceneCameraCutTrack` section range comes back in **display frames** (`end:120`) — in the same payload. This forced the caller to spend narration reconciling that both spans are actually 5s (SAY: "the camera-cut section still reads {start:0, end:120} — hmm, its range is displayed in display frames (0-120) while the transform section reads in tick units (0-120000)... a units display difference"). A consumer verifying "keyframes at distinct frames" cannot compare a key at frame `120000` against a section ending at frame `120` without knowing which unit each is in. This is distinct from `#1`/`#2` (which are the get_properties/set_properties/set_view_range PLAYBACK fields) — here the drift is BETWEEN sibling sections WITHIN one `list_sections` payload, so `list_sections` is now added to the affected-methods/tags. Proposed fix: normalize all frame numbers in `list_sections` to one unit (prefer display frames, matching the Sequencer UI) OR emit an explicit per-value unit tag / a top-level `{frameRate,tickResolution}` so readers can convert. Downstream wiki page: `docs/wiki-src/sequencer.md` (`list_sections` H3). Same underlying frame-vs-tick root as `#1`/`#2`; no calls errored (all 31 ok=true).
- `#2-additional-set-properties-is-ticks-not-frames` `OPEN` reporter — Additional evidence (new angle, source-confirmed + live-replayed on `/Game/Cinematics/CS_ReplayUnits`): the original `#1` claim that `set_properties` takes `playbackStart`/`playbackEnd` "in frames" is WRONG — it takes **raw tick-resolution units**, while the param schema advertises them as display frames. `SequenceHandler.cpp:497,499` builds `StartFrame = FFrameNumber(static_cast<int32>(PlaybackStartValue))` / `EndFrame = FFrameNumber(static_cast<int32>(PlaybackEndValue))` — a bare cast into `FFrameNumber` with **no** `FFrameRate::TransformTime(displayRate → tickResolution)` conversion (contrast the keyframe path's `SequenceHelpers::DisplayFrameToTick`, `SequenceHandler.cpp:225-231`, which *does* convert). Yet `SequenceHandler.cpp:435-436` documents the params as `RPC_PARAM_OPT("playbackStart", "number", "Playback start frame")` / `"Playback end frame"`. Verbatim repro at 24 fps (tickResolution 24000/1): `sequencer.set_properties {path, playbackStart:0, playbackEnd:120}` → `{"playbackStart":0,"playbackEnd":120,"duration":120,"applied":true}` — a caller intending "frame 120 = 5 s" silently gets a **120-tick = 0.005 s** range (a ~1000× error) with `applied:true` and no warning. Getting a real 5 s range requires `playbackEnd:120000` (raw ticks), confirmed: `set_properties {playbackEnd:120000}` → `duration:120000`, and `get_properties` echoes `playbackStart:0, playbackEnd:120000, duration:120000, tickResolution:{24000,1}` — i.e. `get_properties` is **also ticks here**, consistent with `set_properties`, NOT the display-frame flip `#1` observed (so the readback unit may itself vary by path/asset — reinforcing the drift). The mislabeled `"frame"` param is the concrete quotable defect: fix is (a) rename/annotate the params as tick-resolution units OR actually convert display frames → ticks inside the handler (using `DisplayFrameToTick` like the keyframe path), and (b) annotate `get_properties`/`set_properties`/`set_view_range` units in `docs/wiki-src/sequencer.md`. Encountered in a fresh establishing-shot authoring task (build CS_Establishing 24fps 0-5s); outcome ergo, culprit `sequencer.set_properties`.
