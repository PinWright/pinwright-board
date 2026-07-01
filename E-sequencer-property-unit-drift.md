---
id: E-sequencer-property-unit-drift
title: "sequencer playback fields shift units (ticks vs display-frames vs seconds) across sibling readers/writers"
status: OPEN
severity: Low
category: ergonomic
tags: [sequencer, units, docs, get_properties, set_properties, set_view_range]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
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
