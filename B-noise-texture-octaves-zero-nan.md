---
id: B-noise-texture-octaves-zero-nan
title: "texture.create_noise_texture octaves:0 divides by a zero MaxValue and produces NaN pixels"
status: IN-REVIEW
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
- `#2-octaves-range-refused` `IN-REVIEW` developer -- Defect confirmed, symptom corrected: the 0/0
  NaN is real, but it does **not** reach the buffer as a NaN. `FMath::Clamp` is
  `Max(Min(X, Hi), Lo)` with `Max(A,B) = (B<A)?A:B` / `Min(A,B) = (A<B)?A:B`, and every comparison
  against a NaN is false, so `Clamp(NaN, 0, 1)` returns `1` -- verified identical in UE 5.3 and 5.8
  headers. `octaves:0` therefore shipped a uniformly white texture with an unqualified success,
  which is the worse failure because nothing distinguishes it from a legitimate output. Fixed by
  refusing, not substituting: `create_noise_texture` in `Handlers/Material/TextureHandler.cpp` now
  rejects `octaves` outside 1-16 before anything is created, through the file's existing
  `TEXTURE_ERROR_RESPONSE` macro (no new `ERR_*` constant; the file hand-spells literals and must
  stay non-adopting), with the bounds as `MinNoiseTextureOctaves` / `MaxNoiseTextureOctaves` beside
  `FBMNoise`. The upper bound is scope beyond the reported zero and is called out for review: it
  also removes an unbounded per-pixel loop a caller could drive to an editor hang. `octaves` param
  description now states the range. New test
  `PinWright.texture.create_noise_texture.OctavesRangeIsEnforced` in
  `Tests/Material/TestNoiseTextureFormatAndOctaves.cpp` asserts 0 / -3 / 17 are refused naming the
  range and the value passed, that no asset is left behind (so no laundered-NaN buffer exists), and
  that the `octaves:1` boundary produces a finite, varying half-float field. Adjacent and not fixed:
  a caller can still reach `MaxValue == 0` with `persistence: -1` and an even octave count.
