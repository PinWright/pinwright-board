---
id: F-post-process-typed-setters
title: "Typed PostProcessVolume setters (color grading, bloom, Lumen GI/reflections, AA, motion blur)"
status: DONE
severity: Low
category: feature
tags: [post-process, lighting, volume, property-set, boverride-foot-gun]
---

# Typed PostProcessVolume setters (color grading, bloom, Lumen GI/reflections, AA, motion blur)

`volume.add_post_process_volume` / `volume.create_post_process_volume` spawn the volume, but every knob inside `FPostProcessSettings` (~150 UPROPERTY entries) is only reachable through raw `property.set` calls. Two problems with the raw path:

1. **`bOverride_X` foot-gun.** `FPostProcessSettings` pairs each value field with a `bOverride_*` boolean. Writing the value without flipping the override silently no-ops at composition time — the volume keeps the cinematic default. Callers using `property.set` for `ColorGradingIntensity` have to remember to also `property.set` `bOverride_ColorGradingIntensity=true`, doubling every call and producing confusing "nothing changed" reports.
2. **Discovery cost.** Authors don't know which of the ~150 settings matter for the common ask ("warm up the scene", "punchy bloom", "disable motion blur"). Existing `lighting.set_exposure` and `lighting.set_ambient_occlusion` (see `Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp:1443-1504`) already model the right shape: find-or-spawn an unbound PPV, flip the right `bOverride_*` flags, set the values, return `AddActorVerification`.

**Fix:** Add a small batch of typed setters mirroring the `lighting.set_*` shape, each accepting 3–6 of the most-used knobs and flipping the matching `bOverride_*` flag whenever a value is provided:

- `post_process.set_color_grading` — `whiteTemp`, `tint`, `saturation`, `contrast`, `gamma`, `gain` (global only; shadow/midtone/highlight splits are deferred).
- `post_process.set_bloom` — `intensity`, `threshold`, `method` (`Standard` | `Convolution`), `convolutionScatterDispersion`.
- `post_process.set_lumen_gi` — `enabled` (toggles `DynamicGlobalIlluminationMethod` to `Lumen` / `None`), `sceneDetail`, `finalGatherQuality`, `maxTraceDistance`.
- `post_process.set_lumen_reflections` — `enabled` (toggles `ReflectionMethod` to `Lumen` / `ScreenSpace`), `quality`, `rayLightingMode`, `maxRoughnessToTrace`.
- `post_process.set_anti_aliasing` — `method` (`None` | `FXAA` | `TAA` | `TSR` | `MSAA`), `screenPercentage` (writes the `r.ScreenPercentage` companion via `IConsoleManager` if the field isn't on the volume).
- `post_process.set_motion_blur` — `amount`, `max`, `targetFps`, `perObjectSize`.

All six share the find-or-spawn-unbound-PPV preamble from `lighting.set_exposure`; factor it into a small static helper in this handler file (don't lift into a shared util until a third caller appears). Each setter must set `bOverride_<Field>=true` whenever it writes `<Field>`, mirroring the `lighting.set_ambient_occlusion` pattern.

Place the handlers in a new `Source/PinWright/Private/Handlers/Environment/PostProcessHandler.cpp` next to `LightingHandler.cpp`. Add an editorial overlay page at `docs/wiki/post_process.md` summarising the override-flag foot-gun and pointing at `property.set` for everything beyond the curated knob set. No need to deprecate `property.set` on PPV — it remains the escape hatch for advanced fields not covered by typed setters.

Out of scope (deferred until requested):
- Per-channel color grading splits (`Shadows*`, `Midtones*`, `Highlights*`).
- Depth-of-field, chromatic aberration, vignette, film grain, screen-space reflections (legacy), path tracer, ray-traced shadows.
- Reading current values back; `asset.dump` on the level already covers that.

## History
- `#1-initial-proposal` `OPEN` reporter — Raw `property.set` on `FPostProcessSettings` fields silently no-ops unless the paired `bOverride_*` boolean is also set; ~150 settings are otherwise undiscoverable without engine source knowledge. Proposed: six typed setters (`color_grading`, `bloom`, `lumen_gi`, `lumen_reflections`, `anti_aliasing`, `motion_blur`) mirroring `lighting.set_exposure` / `lighting.set_ambient_occlusion`, each flipping the right override flags. New file `Handlers/Environment/PostProcessHandler.cpp`; advanced fields still reachable via `property.set`.
- `#2-post-process-typed-setters` `IN-REVIEW` developer — Added Handlers/Environment/PostProcessHandler.cpp with 6 typed setters (color_grading, bloom, lumen_gi, lumen_reflections, anti_aliasing, motion_blur) following the lighting.set_ambient_occlusion pattern: file-local FindOrSpawnUnboundPPV helper + per-knob value + bOverride_<Field> flip. set_anti_aliasing writes the r.AntiAliasingMethod and r.ScreenPercentage CVars (no per-volume override exists for these). Color grading saturation/contrast/gamma/gain accept scalar (broadcast to FVector4) or [r,g,b,luminance]. Regression test FPostProcessSetBloomFlipsOverrideFlagTest asserts both the value AND the bOverride flag — directly guards the foot-gun.
- `#3-skip-editor-offline` `SKIP` tester — Editor at 127.0.0.1:19880 not running (connection refused, no UnrealEditor.exe process); cannot exercise post_process.* RPCs or run the regression test. Source inspection of PostProcessHandler.cpp shows all 6 handlers registered, each value-write paired with bOverride_<Field>=true, shared FindOrSpawnUnboundPPV helper in PostProcessVolumeUtils.h, set_anti_aliasing uses IConsoleManager CVars as described. Needs live verification once editor is up.
- `#4-verify-bloom-override-flip` `DONE` tester — Verified: schema for `post_process.set_bloom?` lists intensity/threshold/method/convolutionScatterDispersion. Called `post_process.set_bloom {intensity:2.5, threshold:-1.0}` against live editor (spawned/used PostProcessVolume2 in L_Core). Read-back via `property.get`: Settings.BloomIntensity=2.5, Settings.bOverride_BloomIntensity=true, Settings.BloomThreshold=-1, Settings.bOverride_BloomThreshold=true. Foot-gun guard confirmed — each value write paired with matching bOverride_* flip.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 2 body citations repointed in place and verified against plugin HEAD `ef8a1f1b`. `:820-969` had drifted onto `ALightmassImportanceVolume` brush geometry and volumetric-fog colour handling. The two exemplars the body points at are `lighting.set_exposure` at `:1443-1504` (`bOverride_*` at `:1469`/`:1475`/`:1483`, `AddActorVerification` `:1501`) and `lighting.set_ambient_occlusion` at `:1506-1637`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
