---
id: B-effect-debug-options-ignored
title: "effect.draw_debug_shape reports success while ignoring scale, autoDestroy, and plane boxSize"
status: DONE
severity: Medium
category: bug
tags: [effect, debug-shape, accepted-and-ignored, scale, auto-destroy, plane]
---

# Three documented controls never affect the draw call

`effect.draw_debug_shape` documents `scale`, `autoDestroy`, and `boxSize` for box/plane shapes
(`EffectHandler.cpp:222-239`). The handler parses `scale` into `ScaleArr` and `autoDestroy` into
`bAutoDestroy` (`:261-280`), but neither variable is used afterward. Every `DrawDebug*` call passes
the same hard-coded `bPersistentLines=false`. In the plane branch, the `boxSize` condition contains
only `// parsing placeholder` (`:438-444`), so a requested non-square plane is silently drawn at
the scalar default size.

The response always reports success, shape type, location, and duration, with no applied option
readback (`:465-472`). This makes the ignored requests indistinguishable from applied ones.

## What it should do

Apply scale consistently to shape dimensions; parse plane `boxSize` as the box branch already does;
and map/document `autoDestroy` to real lifetime/persistence semantics. Echo the effective geometry
and lifetime in the result, or reject controls unsupported by a selected shape.

## Workaround

Use the scalar `size` field and assume duration-based, non-persistent lines; there is no workaround
for an independently sized plane through this verb.

## Related

- `F-effect-draw-debug-shapes-batch`

## History
- `#1-source-scan` `OPEN` reporter -- Confirmed from full handler control flow; this is not the
  explicitly documented unused legacy `preset` field.
- `#2-options-applied-or-refused` `IN-REVIEW` developer -- Still reproduced on the current tree
  (`ScaleArr`/`bAutoDestroy` unused, plane `boxSize` branch empty, every draw
  `bPersistentLines=false`). Fixed in `Source/PinWright/Private/Handlers/VFX/EffectHandler.cpp`:
  `scale` multiplies box/plane half-extent per axis and every other shape's dimensions by one
  uniform factor (non-uniform refused; line refuses scale); plane parses `boxSize` through the same
  validated reader as box (exactly 3 finite non-negative numbers); `autoDestroy` true (default,
  what the verb always did) = expires after `duration`, false = persistent lines until
  `effect.clear_debug_shapes`, and `duration` with false is refused. A per-shape control table
  refuses with `INVALID_ARGUMENT` any shape control the shape does not draw with (also covers the
  previously ignored `rotation` on sphere/circle/etc., `color` on coordinate, `thickness` on point,
  `boxSize`/`endLocation`/cone/capsule keys on other shapes), plus `size`+`boxSize` together and a
  negative duration. Box now honours `rotation`. Response echoes `scale`, `geometry` (effective
  values handed to DrawDebug*), `autoDestroy`, `persistent`, and `duration` only when timed.
  Tests in `Source/PinWright/Private/Tests/Assets/TestVFXHandlers.cpp` read the world's persistent
  line batcher directly: `PinWright.effect.draw_debug_shape.ScaleAndBoxSizeReachDrawnLines`,
  `.AutoDestroyMapsToLineLifetime`, `.RefusesIgnoredControls` (filter
  `PinWright.effect.draw_debug_shape`). Docs: `docs/wiki-src/effect.md` `## Debug shapes`,
  CHANGELOG. Follow-up for malformed color/rotation/location vectors:
  `B-effect-debug-shape-malformed-vectors-default`.
- `#3-review-scalar-validation` `IN-REVIEW` developer -- Review follow-up: `size`, `thickness`,
  `length`, `angle` and `halfHeight` now refuse negative / non-finite values with
  `INVALID_ARGUMENT` before drawing, like `scale` / `boxSize`; five cases added to
  `PinWright.effect.draw_debug_shape.RefusesIgnoredControls`. The test-side
  `UWorld::GetLineBatcher` 5.6 guard is recorded in `docs/engine-version-support.md`.
- `#4-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full, non-skipped, all reading the world's line batcher: `PinWright.effect.draw_debug_shape.ScaleAndBoxSizeReachDrawnLines` (scale and plane boxSize change the drawn geometry), `.AutoDestroyMapsToLineLifetime` (autoDestroy true expires after duration; false draws persistent lines) and `.RefusesIgnoredControls` (controls a shape does not use, size+boxSize together, and negative or non-finite scalars are refused with INVALID_ARGUMENT). `.ValidParamsNoCrash` also passed. Every ask is met: scale is applied, plane boxSize is parsed, autoDestroy maps to real lifetime, and the result echoes scale, geometry, autoDestroy, persistent and duration while unsupported controls are refused. Malformed vectors are tracked separately in B-effect-debug-shape-malformed-vectors-default.
