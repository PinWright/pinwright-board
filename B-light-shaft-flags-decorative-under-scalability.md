---
id: B-light-shaft-flags-decorative-under-scalability
title: "A directional light's light-shaft flags are decorative whenever [PostProcessQuality@1] has zeroed r.LightShaftQuality, and no read-back distinguishes that from a working setup"
status: IN-REVIEW
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
- `#2-measured-light-shaft-verb` `IN-REVIEW` developer — Premise verified against UE 5.8 source before
  fixing, and it holds in a stronger form than the ticket states: `r.LightShaftQuality` is a binary
  master switch, not a quality dial (`LightShaftRendering.cpp:18-25`, `int32 GLightShafts = 1`,
  `ECVF_Scalability | ECVF_RenderThreadSafe`), and its single read sits in
  `ShouldRenderLightShafts(ViewFamily)` (`:110-120`), which is the opening test of BOTH
  `FDeferredShadingSceneRenderer::RenderLightShaftOcclusion` (`:436`) and `::RenderLightShaftBloom`
  (`:541`) — **above** the loop over `Scene->Lights`, so at 0 the component flags are never read at
  all. Mobile sun-shafts die the same way (`SceneVisibility.cpp:5688`). `BaseScalability.ini` zeroes it
  in `[PostProcessQuality@0]` (:580) **and** `[PostProcessQuality@1]` (:602), restoring it at `@2`
  (:637) / `@3` (:674) / `@Cine` (:714) — the ticket named only `@1`. Unlike `r.LightFunctionQuality`
  and `r.ShadowQuality`, this cvar has no `EngineShowFlagOverride` entry (`ShowFlags.cpp:448-518`), so
  the editor's `LightShafts` show flag stays lit while nothing renders.
  **There was no light-shaft verb to fix** — `grep LightShaft` over `Source/` returned only the wiki
  prose. `lighting.spawn_light` never touches the flags, so the only write path was generic
  `property.set`, which publishes nothing. Added `lighting.setup_light_shafts` in
  `Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp` carrying the house pattern from
  `setup_volumetric_fog`: `bloomEnabled` / `occlusionEnabled` are measured as
  `componentFlag && cvar != 0`, `componentFlags {bloom, occlusion}` are read back off the component
  (never echoed from the request), `lightShaftQualityCVar {cvar, found, value}` carries the measurement
  with `value` OMITTED when the registry lacks the cvar, and `cvarWarning` names the remedy —
  `system.console_command "r.LightShaftQuality 1"`, `sg.PostProcessQuality 2`, or `[SystemSettings]` in
  `DefaultEngine.ini`. It does NOT set the cvar. Two further silent no-ops closed at the same time:
  the verb targets by `UDirectionalLightComponent` and refuses a non-directional light outright
  (`ShouldRenderLightShaftsForLight`, `:122-130`, returns false for every other type, so
  `bEnableLightShaftBloom` on a point/spot/rect light is inert despite the engine tooltip), and it
  refuses a registered Static-mobility component up front because both engine setters no-op there
  (`USceneComponent::AreDynamicDataChangesAllowed`, `SceneComponent.h:1363`). It also errors on an
  ambiguous target rather than writing to whichever sun the level lists first, and errors when neither
  `bloom` nor `occlusion` is named rather than inventing a write. Tests:
  `PinWright.lighting.setup_light_shafts.EnabledFollowsCVarVeto` (differential — the same call at
  `r.LightShaftQuality` 0 and 1 must disagree on `bloomEnabled` / `occlusionEnabled` while
  `componentFlags` reads true in both; cvar written at its existing SetBy priority and restored) and
  `PinWright.lighting.setup_light_shafts.RefusesRequestWithNothingToWrite`, both in
  `Source/PinWright/Private/Tests/Environment/TestLightShaftCVarVeto.cpp`. No new error codes.
  **Audit of the rest of `lighting.*` — named, not fixed here.** (1) `lighting.configure_shadows`
  echoes `virtualShadowMaps` straight off the request and reports `success: true` even when
  `FindConsoleVariable("r.Shadow.Virtual.Enable")` returns null or a higher-priority setter refuses the
  write — the accepted-and-ignored-parameter shape, worse than this ticket's because nothing is
  measured at all. (2) `lighting.spawn_light`'s `castShadows` is vetoed wholesale by
  `r.ShadowQuality=0` in `[ShadowQuality@0]`, which drives `EngineShowFlags.SetDynamicShadows(false)`
  (`ShowFlags.cpp:490-496`). (3) `lighting.set_ambient_occlusion` is vetoed by
  `r.AmbientOcclusionLevels=0` in `[PostProcessQuality@0]`. (4) `lighting.spawn_sky_light`'s intensity
  is silently SCALED, not vetoed, by `r.SkylightIntensityMultiplier=0.8` in
  `[GlobalIlluminationQuality@0]` (`SkyLightComponent.cpp:85-91`) — the echoed value is not the
  effective one. (5) `lighting.set_exposure` was checked and is NOT in this class:
  `r.EyeAdaptationQuality=2` at every `PostProcessQuality` level. Also outside `lighting.*` but the
  same shape: `r.DistanceFieldShadowing=0` (`[ShadowQuality@0/@1]`) vetoes
  `ULightComponent::bUseRayTracedDistanceFieldShadows`, `r.CapsuleShadows=0` vetoes
  `bCastCapsuleDirectShadow` / `bCastCapsuleIndirectShadow`, and `r.LightFunctionQuality=0` vetoes
  `LightFunctionMaterial`.
