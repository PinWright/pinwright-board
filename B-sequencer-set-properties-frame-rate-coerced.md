---
id: B-sequencer-set-properties-frame-rate-coerced
title: "sequencer.set_properties silently rounds and clamps frameRate (29.97 -> 30, 2000 -> 960)"
status: OPEN
severity: Medium
category: bug
tags: [sequencer, frame-rate, validation, coercion, false-success]
encounters: 1
lastSeen: 2026-10-02T00:00:00+03:00
rice: [1, 3, 1, 1]
priority: 33
---

# set_properties coerces the display rate instead of refusing it

## What happens

`sequencer.set_properties` (`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`,
the `bHasFrameRate` block) refuses only `frameRate <= 0`, then writes
`FFrameRate(FMath::Clamp(FMath::RoundToInt(FrameRateValue), 1, 960), 1)` and reports success.
`29.97` is stored as `30/1`, `2000` as `960/1`, with no signal in the response. There is no
way to set a rational (NTSC) rate through this verb.

## What should happen

Same contract as the fixed sibling `sequencer.set_display_rate`
(`B-sequencer-display-rate-invalid`): parse strictly (reuse `SequenceHandlerDisplayRate::TryParse`
in the same file), refuse non-integral / out-of-range values with `INVALID_ARGUMENT` before any
write, accept the `"N/D"` form, and echo the stored rate.

## Workaround

Use `sequencer.set_display_rate` for the rate and `set_properties` only for the playback range.

## History
- `#1-filed-sibling-scan` `OPEN` developer — Found while fixing `B-sequencer-display-rate-invalid`; source read only, no repro run. Dedupe: board grep for `set_properties` + frameRate/clamp/round found only frame-vs-tick unit tickets (`E-sequencer-property-unit-drift`, `B-sequencer-section-range-display-frames-as-ticks`), none covering the rate coercion.
