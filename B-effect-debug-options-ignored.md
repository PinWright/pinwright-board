---
id: B-effect-debug-options-ignored
title: "effect.draw_debug_shape reports success while ignoring scale, autoDestroy, and plane boxSize"
status: OPEN
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
