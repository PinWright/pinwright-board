---
id: B-noise-texture-octaves-zero-nan
title: "texture.create_noise_texture octaves:0 divides by a zero MaxValue and produces NaN pixels"
status: OPEN
severity: High
category: bug
tags: [texture, create_noise_texture, octaves, divide-by-zero, nan, unvalidated-input]
encounters: 1
lastSeen: 2026-08-27
---

# An unvalidated zero produces a texture of NaN

`texture.create_noise_texture` with `octaves: 0` runs the FBM accumulation zero times, leaving
`MaxValue` at zero, and then normalises by it. Every pixel comes out NaN.

`0` is not an absurd thing for a caller to pass -- it reads as "no octaves of detail", i.e. flat --
and nothing rejects it.

**Fix:** reject `octaves: 0` naming the valid range, or define it as one octave. Rejecting is the
better half of that: a zero-octave FBM has no meaningful definition, and a verb that quietly
substitutes 1 is back to the accepted-and-ignored shape the sibling `noiseType` defect had.

Found while fixing the two noise defects in the same function and deliberately left alone as out of
scope.

## History
- `#1-found-in-the-same-function` `OPEN` reporter -- Recorded by the agent fixing
  `B-noise-texture-noisetype-ignored` and `B-noise-texture-seamless-lattice`. Source-level claim.
