---
id: B-noise-texture-octaves-zero-nan
title: "texture.create_noise_texture octaves:0 divides by a zero MaxValue and produces NaN pixels"
status: DONE
severity: High
category: bug
tags: [texture, create_noise_texture, octaves, divide-by-zero, nan, unvalidated-input]
encounters: 1
lastSeen: 2026-08-28
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

- `#3-verified-behaviourally-on-the-built-binary` `DONE` verifier — 2026-08-28, against the
  `b79ba53e` build. `octaves: 0` is refused before anything is created:
  `[TEXTURE_ERROR] octaves must be between 1 and 16, got 0. An FBM with no octaves has no value to
  normalise by, and beyond 16 an octave can no longer change a pixel.` The message names the range
  AND the value passed, as `#2` claimed. Reading the target path back afterwards returns
  `[ASSET_NOT_FOUND]`, so **no asset is left behind** and the laundered-NaN white buffer cannot
  exist. `octaves: 17` is refused with the same sentence, so the upper bound `#2` flagged for review
  is live. `octaves: 1` and `octaves: 3` both produce ordinary varying fields (the `octaves:3`
  256x256 control measures mean 130.70, min 37, max 219, `grayscale: true`, hash `ff5990de`) — the
  bound refuses only what it says it refuses. This is a refusal, not a substitution, which is what
  the ticket asked for. Closing. The adjacent case `#2` recorded as still reachable —
  `MaxValue == 0` via `persistence: -1` with an even octave count — was not exercised here and is
  not covered by this closure.
