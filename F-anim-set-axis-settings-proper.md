---
id: F-anim-set-axis-settings-proper
title: "Reimplement animation.authoring.set_axis_settings to actually write BlendSpace axis name/min/max/grid (removed as a no-op stub)"
status: OPEN
severity: Medium
category: feature
tags: [animation, blend-space, set_axis_settings, authoring, reimplement, rpc-cull]
---

# Reimplement `animation.authoring.set_axis_settings` properly (BlendSpace axis config)

`animation.authoring.set_axis_settings` was **removed** in the RPC cull recorded
in [`E-rpc-cull-151-record`](E-rpc-cull-151-record.md) because it was a no-op
stub. The capability — edit a `UBlendSpace`'s axis name, min/max range, and grid
division count **after** creation — is legitimately wanted: axis config is the
coordinate space that blend samples live in, and today it can only be set at
create time (and `create_blend_space_1d` does not even expose `gridDivisions`).

## What the removed version did wrong

The old handler (`Handlers/Animation/AnimationAuthoringHandler_BlendSpace.cpp:580-599`)
read `axisName`/`minValue`/`maxValue`/`gridDivisions` into locals that were
**never applied to anything**, then called `PostEditChange()` +
`MarkPackageDirty()` + `SaveAnimAsset` on the unchanged asset and returned
`success:true` "Axis settings updated". The handler's own comment admitted it:
`"skip direct modification since BlendParameters is protected ... note it may
not take effect in UE 5.7+"`. Reporters who thought it "worked" were actually
seeing `create_blend_space_1d`'s hardcoded `GridNum = 4` from creation — the
`set_axis_settings` call applied nothing.

## Proper implementation

The write is already solved elsewhere in the same file. Use the shared
reflection helper `AnimationAuthoringHelpers::GetBlendParametersForWrite` (the
protected-member workaround that `create_blend_space_1d` (line 350),
`create_blend_space_2d` (line 453), and `AnimationHandler.cpp
ApplyBlendSpaceConfiguration` (line 199) already use) to write the requested
axis's `DisplayName`, `Min`, `Max`, and `GridNum` onto the resolved
`FBlendParameter`, then `PostEditChange` + save. Resolve the target axis by name
or index (0 = horizontal, 1 = vertical for 2D). Fail loud on an out-of-range
axis index rather than silently succeeding.

**Fix:** New handler in `AnimationAuthoringHandler_BlendSpace.cpp` (6 other real
handlers remain in that file). Regression test: create a blend space, run
`set_axis_settings` with a distinct name/min/max/grid, and assert the written
`BlendParameters[axis]` fields via `asset.dump`/readback (the removed stub would
leave them unchanged — differential proof).

## Cross-references

- [`E-blend-space-grid-divisions-on-axis-settings-undiscoverable`](E-blend-space-grid-divisions-on-axis-settings-undiscoverable.md)
  is deferred behind `B-create-blend-space-axis-config-dropped-on-57`; its `#3`
  note already established that `set_axis_settings` writes nothing and that the
  genuine remaining work is the code fix here. That docs ticket cannot honestly
  point at a working knob until this method exists again.
- [`B-create-blend-space-axis-config-dropped-on-57`](B-create-blend-space-axis-config-dropped-on-57.md)
  shares the same axis-config FProperty-reflection write surface.

## History
- `#1-reimpl-after-cull` `OPEN` reporter — Filed to reinstate the wanted capability removed by the RPC cull ([`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)). The removed `animation.authoring.set_axis_settings` read axisName/min/max/gridDivisions into locals it never applied (AnimationAuthoringHandler_BlendSpace.cpp:580-599), then saved the unchanged asset and returned a false "Axis settings updated". Proper impl: write DisplayName/Min/Max/GridNum via the existing `AnimationAuthoringHelpers::GetBlendParametersForWrite` reflection helper already used by the create_blend_space handlers. Cross-refs the deferred docs ticket `E-blend-space-grid-divisions-on-axis-settings-undiscoverable` (whose #3 note flagged this method as a no-op) and `B-create-blend-space-axis-config-dropped-on-57` (same axis-config surface).
