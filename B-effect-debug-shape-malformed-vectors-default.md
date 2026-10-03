---
id: B-effect-debug-shape-malformed-vectors-default
title: "effect.draw_debug_shape replaces a malformed color/rotation/location vector with its default and reports success"
status: OPEN
severity: Low
category: bug
tags: [effect, debug-shape, accepted-and-ignored, validation]
---

# Malformed vector controls fall back silently

After `B-effect-debug-options-ignored`, `scale` and `boxSize` are validated (exactly three finite
non-negative numbers, else `INVALID_ARGUMENT`). The remaining vector controls of
`effect.draw_debug_shape` in `Source/PinWright/Private/Handlers/VFX/EffectHandler.cpp` still fall
back without a word:

- `color` with fewer than 3 elements (or non-numbers) draws white; components outside 0-255 wrap
  through the `(uint8)` cast.
- `rotation` with fewer than 3 elements is read as zero by `ParseRotationArray`.
- `location`, `endLocation` and `direction` with fewer than 3 array elements (or an object
  missing axes) use the default / partially default vector via `ParseLocationFromPayload`.

The response then echoes the substituted values, so the caller can only notice by comparing.

## What it should do

Refuse a malformed vector with `INVALID_ARGUMENT` naming the key and the accepted shape, as
`scale` / `boxSize` now are. `ParseLocationFromPayload` is also used by `effect.spawn_niagara` in the same file, so check
that caller before tightening the shared helper.

## History
- `#1-found-while-fixing` `OPEN` developer -- Found while fixing B-effect-debug-options-ignored;
  left out of that change to keep it to the ticket's three controls plus the per-shape refusal.
