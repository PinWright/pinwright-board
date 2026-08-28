---
id: B-lighting-writes-vetoed-by-scalability-cvars
title: "Three more lighting writes are vetoed or scaled by a scalability cvar they do not read"
status: IN-REVIEW
severity: Medium
category: bug
tags: [lighting, scalability, cvar, measured-vs-requested, silent-noop, audit]
encounters: 1
lastSeen: 2026-08-28
---

# The rest of the audit, in one ticket because the fix is one pattern

After `lighting.setup_volumetric_fog` and `lighting.setup_light_shafts` were converted to
measured-vs-requested reporting, an audit of the namespace found three more instances plus one
variant:

- **`lighting.spawn_light`'s `castShadows`** — vetoed wholesale by `r.ShadowQuality=0` in
  `[ShadowQuality@0]`, which drives `EngineShowFlags.SetDynamicShadows(false)` (`ShowFlags.cpp:490-496`).
- **`lighting.set_ambient_occlusion`** — vetoed by `r.AmbientOcclusionLevels=0` in
  `[PostProcessQuality@0]`.
- **`lighting.spawn_sky_light`** — a **scaling** variant rather than a veto:
  `r.SkylightIntensityMultiplier=0.8` in `[GlobalIlluminationQuality@0]`
  (`SkyLightComponent.cpp:85-91`) means the echoed intensity is not the effective one. Worth handling
  because it is the shape that will recur: a cvar that does not disable the write but changes what it
  means.

**Checked and explicitly cleared — do not file:** `lighting.set_exposure`. `r.EyeAdaptationQuality` is
2 at every `PostProcessQuality` level.

Same shape outside `lighting.*`, listed so the next sweep starts here rather than from scratch:
`r.DistanceFieldShadowing=0` (`[ShadowQuality@0/@1]`) vetoes `bUseRayTracedDistanceFieldShadows`;
`r.CapsuleShadows=0` vetoes `bCastCapsuleDirectShadow` / `bCastCapsuleIndirectShadow`;
`r.LightFunctionQuality=0` vetoes `LightFunctionMaterial`.

## History
- `#1-lighting-namespace-audit` `OPEN` reporter — Produced by the audit the
  `B-light-shaft-flags-decorative-under-scalability` agent ran after fixing its own ticket, which was
  asked to name the rest rather than fix them. Source readings against `BaseScalability.ini` and the
  engine's cvar reads; not reproduced.
- `#2-measured-vs-requested-for-all-three` `IN-REVIEW` developer — "Changed lighting.spawn_light, lighting.spawn_sky_light and lighting.set_ambient_occlusion in LightingHandler.cpp to measure the scalability cvar they were silently losing to and report it: spawn_light now publishes castShadows / shadowsEnabled / shadowQualityCVar, spawn_sky_light publishes intensity / effectiveIntensity / skylightIntensityMultiplierCVar, and set_ambient_occlusion's `enabled` became (applied intensity > 0 AND r.AmbientOcclusionLevels != 0) alongside a new ambientOcclusionLevelsCVar; each emits a cvarWarning naming the remedy, and none changes the cvar."
