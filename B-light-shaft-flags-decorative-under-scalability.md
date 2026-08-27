---
id: B-light-shaft-flags-decorative-under-scalability
title: "A directional light's light-shaft flags are decorative whenever [PostProcessQuality@1] has zeroed r.LightShaftQuality, and no read-back distinguishes that from a working setup"
status: OPEN
severity: High
category: bug
tags: [lighting, light-shaft, scalability, cvar, silent-noop, measured-vs-requested, directional-light]
encounters: 1
lastSeen: 2026-08-27
---

# `r.LightShaftQuality` vetoes light shafts the same way `r.VolumetricFog` vetoed volumetric fog

`BaseScalability.ini` sets `r.LightShaftQuality=0` in `[PostProcessQuality@1]`. While it is zero, a
directional light's `bEnableLightShaftBloom` and `bEnableLightShaftOcclusion` render nothing — but the
component flags read back `true`, every write reports success, and nothing published by any PinWright
verb says the pass is switched off.

This is the identical shape to `B-setup-volumetric-fog-enabled-true-while-cvar-off`, which is now
fixed: `lighting.setup_volumetric_fog` reads `r.VolumetricFog` through `IConsoleManager` and reports
`enabled` as `componentFlag && cvar != 0`, with the measured cvar and a warning beside it. The general
rule that fix established: **any component flag whose renderer pass a scalability cvar can switch off
has this failure mode.** Light shafts are the next instance of it.

**Fix:** apply the same measured-vs-requested pattern already shipping in
`Handlers/Environment/LightingHandler.cpp` — read the cvar, report the effective state rather than the
written flag, and name the remedy in a warning. Worth auditing the rest of `lighting.*` for the same
shape at the same time rather than one cvar per ticket.

## History
- `#1-generalised-from-the-fog-fix` `OPEN` reporter — Raised by the agent that fixed
  `B-setup-volumetric-fog-enabled-true-while-cvar-off`, which established the measured-vs-requested
  pattern and noted that the sibling verbs are not yet measured. Source-level claim from
  `BaseScalability.ini` and the light-shaft flags; not reproduced live.
