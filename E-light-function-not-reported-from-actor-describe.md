---
id: E-light-function-not-reported-from-actor-describe
title: "actor.describe and lighting.* report LightFunctionMaterial with nothing beside it, so the atlas verdict is only reachable from the material side"
status: OPEN
severity: Medium
category: ergonomic
tags: [lighting, light-function, atlas, actor-describe, discoverability, cross-reference]
encounters: 1
lastSeen: 2026-08-27
---

# The diagnostic exists, but not on the read the reporter actually made

`B-light-function-atlas-silently-drops-material` is fixed on the material side:
`material.authoring.get_material_info` returns a measured `lightFunctionAtlas` block, and
`material.authoring.set_light_function_atlas_compatible` is the override.

But the read a level author actually makes is `actor.describe` on the light, or a `lighting.*` verb --
and those still report `LightFunctionMaterial` as a bare path with nothing beside it. Someone debugging
"why does my light function not reach the fog" has to already suspect the atlas to find the field that
answers it.

**Fix:** wire `AddReportIfLightFunction` (already written, in
`Handlers/Material/MaterialLightFunctionAtlas.h`) into `actor.describe`'s light-component path, so a
light whose function material cannot enter the atlas says so where the author is looking. Cheap now
that the measurement exists.

## History
- `#1-follow-up-from-the-atlas-fix` `OPEN` reporter -- Raised by the agent that fixed
  `B-light-function-atlas-silently-drops-material`, noting that the files carrying the light-side reads
  were outside its ownership.
