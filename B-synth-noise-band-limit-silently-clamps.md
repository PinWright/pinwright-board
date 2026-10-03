---
id: B-synth-noise-band-limit-silently-clamps
title: "noise generator: lowCutHz / highCutHz above 0.45 * sampleRate are silently clamped by FBiquadFilter, so the band rendered is not the band the recipe names"
status: OPEN
severity: Low
category: bug
tags: [audio, synth, noise, biquad, silent-clamp, sample-rate]
encounters: 1
rice: [1, 2, 1, 1]
priority: 17
---

# Noise band limits ride the biquad's silent clamp

`PwGenOscNoise.cpp` (band-limit block after the colour fill) builds an `Audio::FBiquadFilter`
highpass at `lowCutHz` and lowpass at `highCutHz`. `FBiquadFilter` silently clamps any cutoff to
`0.45 * SampleRate` (Filter.cpp:64-67). The noise rows publish 20..20000 Hz regardless of rate, so:

- `lowCutHz: 5000` at `sampleRate: 8000` renders a highpass at 3600 Hz and reports success;
- `highCutHz: 19950` at 44.1 kHz renders a lowpass at 19845 Hz (inaudible, but still not what was
  asked).

`filter` and `eq` reject the same out-of-window request with INVALID_PARAMS (rpc-design.md §1/§3),
so the noise generator is the odd one out. Found during the sweep for
`B-synth-schema-advertises-unrenderable-ranges`; not visible at the default 48 kHz, where the
window (21600 Hz) covers the whole published range.

## Fix

Reject an APPLIED band limit (one not parked at its row extreme) above `0.45 * sampleRate`, with the
same wording as `PwFxFilter`, and add the rule to the noise kind's published `constraint`.

## History
- `#1-filed` `OPEN` developer — Found while sweeping DSP rejections against the spec table for
  `B-synth-schema-advertises-unrenderable-ranges`.
