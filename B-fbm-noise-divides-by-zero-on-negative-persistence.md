---
id: B-fbm-noise-divides-by-zero-on-negative-persistence
title: "FBMNoise can still divide by zero via a negative persistence with an even octave count"
status: IN-REVIEW
severity: Medium
category: bug
tags: [texture, create_noise_texture, FBMNoise, persistence, divide-by-zero, unvalidated-input]
encounters: 1
lastSeen: 2026-08-28
---

# The other route to the same 0/0

`B-noise-texture-octaves-zero-nan` is fixed by rejecting `octaves` outside 1-16. That closed one route
into `FBMNoise`'s `MaxValue == 0`.

`persistence: -1` with an **even** octave count is another: the amplitude terms alternate sign and sum
to exactly zero, so the normalisation divides by zero again. Whether that produces NaN or (via
`FMath::Clamp`'s NaN-comparison behaviour, see the sibling ticket) a uniformly white image depends on
the same clamp, so the visible symptom is likely the same — a plain white texture reported as success.

`persistence` is currently unvalidated.

**Fix:** validate `persistence` alongside `octaves`. Decide what range is meaningful — a negative
persistence is arguably never intended, and if so refusing it is simpler than defending the sum.

## History
- `#1-adjacent-to-the-octaves-fix` `OPEN` reporter — Found by the agent fixing the octaves defect,
  adjacent to it and out of that ticket's scope. Source reading, not reproduced.
- `#2-persistence-range-enforced` `IN-REVIEW` developer — "Added MinNoiseTexturePersistence/MaxNoiseTexturePersistence (0..1) and a negated-range check beside the octaves check in TextureHandler.cpp, so a negative (or non-finite) persistence is refused before anything is created instead of cancelling the octave amplitudes to zero; documented the range on the persistence param spec and in docs/wiki-src/texture.md, and added PinWright.texture.create_noise_texture.PersistenceRangeIsEnforced"
