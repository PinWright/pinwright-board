---
id: F-anim-set-axis-settings-proper
title: "BlendSpace axis config: no way to rename an axis after creation, and the typed create_blend_space_1d/_2d hardcode GridNum=4 while the wiki says they take grid divisions"
status: OPEN
severity: Low
rice: [1, 2, 1, 2]
priority: 8
category: feature
tags: [animation, blend-space, set_axis_settings, authoring, reimplement, rpc-cull]
---

# BlendSpace axis config: no post-create axis rename, and the typed creators hardcode GridNum=4

`animation.authoring.set_axis_settings` stays removed (RPC cull,
[`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)). Most of what it was meant to do is
now reachable: `animation.create_blend_space` called on an existing blend space of the same
dimensionality re-applies `minX`/`maxX`/`gridX` (and `minY`/`maxY`/`gridY` for 2D) in place
and keeps the samples (`Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp:507`,
update-in-place branch `:592-606`, write in `ApplyBlendSpaceConfiguration` `:192-238`).

Two gaps remain:

1. **No axis rename after creation.** Nothing writes `FBlendParameter::DisplayName` on an
   existing blend space. `animation.create_blend_space` has no axis-name parameter, and
   `ApplyBlendSpaceConfiguration` writes only `Min`/`Max`/`GridNum`
   (`AnimationHandler.cpp:221-234`). The typed creators set the name only at creation.
2. **Typed creators ignore grid.** `animation.authoring.create_blend_space_1d` /
   `_2d` take no grid parameter (`AnimationAuthoringHandler_BlendSpace.cpp:286-296`,
   `:396-409`) and hardcode `GridNum = 4` (`:365`, `:474`, `:482`). The wiki says the
   opposite: `docs/wiki-src/animation.authoring.md:51` states they "take axis min/max and
   grid-division bounds" and that "Blend-space axes are fixed at creation", both false.

**Workaround:** for min/max/grid, call `animation.create_blend_space {name, skeletonPath,
savePath, dimensions, minX, maxX, gridX[, minY, maxY, gridY]}` on the existing asset. Pass
every axis field: an omitted one is reset to its default (min 0, max 1, grid 3), not kept
(`AnimationHandler.cpp:199-202`, `:226-229`). No workaround for renaming an axis.

**Fix:** (a) add `gridDivisions` (1D) / `horizontalGridDivisions` + `verticalGridDivisions`
(2D) to the typed creators and write them instead of the hardcoded 4; (b) add an optional
axis-name parameter (`axisName` / `axisNameY`) to `animation.create_blend_space` and write it
in `ApplyBlendSpaceConfiguration` through the existing
`AnimationAuthoringHelpers::GetBlendParametersForWrite`
(`AnimationAuthoringHelpers.cpp:233`), keeping omitted fields at their current values on
update-in-place; (c) correct `animation.authoring.md:51`.

**Acceptance:** `create_blend_space_1d {gridDivisions:8}` reads back `axes[0].gridNum` 8 in
`animation.authoring.get_animation_info` (`BlendSpaceDumpBuilder.cpp:33-36`); calling `animation.create_blend_space` on that asset
with a new axis name and only `gridX` changes the name and grid and leaves min/max and the
samples unchanged; `animation.authoring.md` no longer claims axes are fixed at creation.

## Cross-references

- [`B-create-blend-space-axis-config-dropped-on-57`](B-create-blend-space-axis-config-dropped-on-57.md)
  shares the same `GetBlendParametersForWrite` write surface.
- [`E-blend-space-grid-divisions-on-axis-settings-undiscoverable`](E-blend-space-grid-divisions-on-axis-settings-undiscoverable.md)
  (WONTFIX) handed its remaining code gap to this ticket.

## History
- `#1-reimpl-after-cull` `OPEN` reporter — Filed to reinstate the wanted capability removed by the RPC cull ([`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)). The removed `animation.authoring.set_axis_settings` read axisName/min/max/gridDivisions into locals it never applied (AnimationAuthoringHandler_BlendSpace.cpp:580-599), then saved the unchanged asset and returned a false "Axis settings updated". Proper impl: write DisplayName/Min/Max/GridNum via the existing `AnimationAuthoringHelpers::GetBlendParametersForWrite` reflection helper already used by the create_blend_space handlers. Cross-refs the deferred docs ticket `E-blend-space-grid-divisions-on-axis-settings-undiscoverable` (whose #3 note flagged this method as a no-op) and `B-create-blend-space-axis-config-dropped-on-57` (same axis-config surface).
- `#2-rephrased` `OPEN` developer — Old text said axis config could only be set at create time and asked to reinstate set_axis_settings wholesale, citing removed-handler lines (AnimationAuthoringHandler_BlendSpace.cpp:580-599) and stale helper call sites. At 7230b41d animation.create_blend_space re-applies min/max/grid in place on an existing asset (AnimationHandler.cpp:592-606), so the ticket is narrowed to the two real gaps: no post-create axis DisplayName write, and no grid param on create_blend_space_1d/_2d (hardcoded GridNum=4 at :365/:474/:482) plus the false animation.authoring.md:51 claim. Added the omitted-field-reset gotcha to the workaround. Severity Medium -> Low: min/max/grid has a workaround and an axis rename is a rare path. rice 1 1 1 1 -> 1 2 1 2: the false animation.authoring.md:51 claim makes it misleading docs (I=2), and the fix spans two creators plus create_blend_space (E=2).
