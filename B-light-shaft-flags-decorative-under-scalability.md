---
id: B-light-shaft-flags-decorative-under-scalability
title: "A directional light's light-shaft flags are decorative whenever [PostProcessQuality@1] has zeroed r.LightShaftQuality, and no read-back distinguishes that from a working setup"
status: DONE
severity: High
category: bug
tags: [lighting, light-shaft, scalability, cvar, silent-noop, measured-vs-requested, directional-light]
encounters: 2
lastSeen: 2026-08-28T09:10:00+05:00
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
- `#3-verified-fixed-behaviourally` `DONE` verifier — 2026-08-28. The verb the fix added exists on the rebuilt binary at HEAD `b79ba53e` and behaves as claimed, verified live on `/Game/Maps/Atlantis` against `Sun_Filtered` (`DirectionalLight_0`, the level's only directional light). Both component flags were read first — `property.get bEnableLightShaftBloom` and `bEnableLightShaftOcclusion` on `…DirectionalLight_0.LightComponent0` both `true` — and the same values were written back, so the level was not changed and was not saved. **Differential, same call either side of a cvar toggle.** At `r.LightShaftQuality 1`: `lighting.setup_light_shafts {actorName:"Sun_Filtered", bloom:true, occlusion:true}` → `{bloomEnabled:true, occlusionEnabled:true, componentFlags:{bloom:true, occlusion:true}, lightShaftQualityCVar:{cvar:"r.LightShaftQuality", found:true, value:1}}`, no warning. At `r.LightShaftQuality 0`, identical call → `{bloomEnabled:false, occlusionEnabled:false, componentFlags:{bloom:true, occlusion:true}, lightShaftQualityCVar:{… value:0}}` plus `cvarWarning` naming `BloomScale` / `BloomThreshold` / `BloomMaxBrightness` / `BloomTint` / `OcclusionMaskDarkness` / `OcclusionDepthRange` / `LightShaftOverrideDirection` as inert, the white dummy occlusion texture, the fact that the `LightShafts` show flag stays lit so the editor UI does not show the veto either, and all three remedies. `componentFlags` reads `true` in both runs while the measured fields disagree, which is the whole point of the ticket. **The veto is real and was seen, not inferred:** identical pose `{x:-10500,y:-3800,z:1500}` / `{pitch:4,yaw:40,roll:0}` / `fov 70`, 448x448, game view on, pinned `exposure {mode:"fixed", ev100:0}` — `verify_shafts_on.png` against `verify_shafts_off.png` shows the tall column's hard dark light-shaft-occlusion band appearing only at cvar 1, with `maxLuminance` 0.649292 vs 0.539691, `luminanceVariance` 0.009528 vs 0.005766, and the mean DARKENING 0.258631 → 0.246327 with shafts on — the same counter-intuitive direction the sibling ticket's encounter `#3` measured (0.616936 → 0.443637), because occlusion is what supplies the contrast between shafts. **The two further silent no-ops the fix closed were confirmed as refusals, not writes:** `{actorName:"Sun_Filtered"}` with neither flag → `[INVALID_ARGUMENT] At least one of bloom or occlusion is required. With neither there is nothing to write, and a success carrying the light's current state would read as a write that happened`; and `{actorName:"Portal_Light", bloom:true}` on the level's point light → `[ACTOR_NOT_FOUND] No actor named 'Portal_Light' with a UDirectionalLightComponent in the level`, rather than the inert `bEnableLightShaftBloom` write the engine tooltip invites. `r.LightShaftQuality` was restored to `1`. The audit items the developer entry named but did not fix (`configure_shadows`, `spawn_light`'s `castShadows`, `set_ambient_occlusion`, `spawn_sky_light`'s scaled intensity) were NOT verified here and remain open work wherever they are tracked.
