---
id: B-setup-volumetric-fog-enabled-true-while-cvar-off
title: "lighting.setup_volumetric_fog reports enabled:true when r.VolumetricFog is 0, so the pass cannot render and every fog value tuned against that frame is tuned against the wrong renderer"
status: DONE
severity: High
category: bug
tags: [lighting, setup-volumetric-fog, exponential-height-fog, cvar, scalability, silent-noop, misleading-success, measured-vs-requested]
encounters: 4
lastSeen: 2026-08-28T09:10:00+05:00
---

# `enabled: true` means "the component flag is set", not "volumetric fog will render"

`lighting.setup_volumetric_fog` sets `bEnableVolumetricFog` on the level's
`AExponentialHeightFog` and answers:

```json
{"success": true, "enabled": true, "actorName": "ExponentialHeightFog",
 "actorPath": "/Game/Maps/Atlantis.Atlantis:PersistentLevel.ExponentialHeightFog_0",
 "actorClass": "ExponentialHeightFog", "existsAfter": true}
```

Verbatim, from this session. There is no volumetric fog in the frame.

While the `r.VolumetricFog` console variable is `0` the component flag is decorative: the
volumetric pass does not run, so `VolumetricFogAlbedo`, `VolumetricFogEmissive`,
`VolumetricFogDistance`, `VolumetricFogExtinctionScale` and
`VolumetricFogScatteringDistribution` are all inert, and every light's
`VolumetricScatteringIntensity` does nothing. `property.get bEnableVolumetricFog` returns
`true` and `property.get VolumetricFogDistance` returns the value that was set, so **no
read-back available to the caller distinguishes this state from a working one.**

## Repro

Identical pose, pinned exposure `ev100 0`, game view on, `hideEditorSprites: true`,
768x768, camera `location {x:-10500,y:-3800,z:1500}` `rotation {pitch:4,yaw:40,roll:0}`
`fov 70`, on a level whose `ExponentialHeightFog` already has `bEnableVolumetricFog: true`
and where `lighting.setup_volumetric_fog {viewDistance: 65000}` has already answered
`enabled: true`:

```js
call("actor.set_component_properties", {                  // VolumetricFogAlbedo -> (130,195,225)
  actorName: "ExponentialHeightFog_0", componentName: "HeightFogComponent0",
  properties: { VolumetricFogAlbedo: {R:130, G:195, B:225, A:255} }})
call("render.capture_open_level", { /* pose above */ })   // meanLuminance 0.194087
call("system.console_command", { command: "r.VolumetricFog 0" })
call("render.capture_open_level", { /* same pose */ })    // meanLuminance 0.194400
call("system.console_command", { command: "r.VolumetricFog 1" })
call("render.capture_open_level", { /* same pose */ })    // meanLuminance 0.383695
```

## Evidence

| call | frame `meanLuminance` |
|---|---|
| `VolumetricFogAlbedo` -> (130,195,225) through the notifying verb | 0.194087 |
| `r.VolumetricFog 0` | 0.194400 |
| `r.VolumetricFog 1` | **0.383695** |

Captures under `Saved/Screenshots/OpenLevel/` on host project EAContentExamples58:
`env_godrays_probe_albedo.png`, `env_probe_rvolfog0.png`, `env_probe_rvolfog1.png`.

Turning the cvar **off** moved the frame by 3e-4 — nothing, because it was already off.
Turning it **on** doubled mean luminance and put volumetric fog in the frame for the first
time in that session. The albedo write immediately before it, through
`actor.set_component_properties` (which does fire `PostEditChangeProperty` — see
`B-set-component-properties-no-change-notification`), also moved the frame by only 7e-5,
for the same reason: it was writing to a pass that was not running.

A probe note worth keeping, because it cost cycles: **`r.VolumetricFog 0` is
indistinguishable from "already 0".** Toggling it off as a diagnostic moved the frame by
3e-4 — nothing — because it was already off. Only turning it *on* proves anything. A probe
that returns the same answer for "no effect" and "already in that state" is not a probe.

The `r.VolumetricFog 1` frame is also the first in that level to contain a **god ray**: a
`SM_Rock_A` occluder 2200 uu above the seafloor casts a visibly darker column of water down
to its ground shadow. In the two frames before it the same rock casts a ground shadow with
no shaft above it. That missing shaft is the whole symptom in one picture, and it is the
acceptance test a fix must reproduce.

## Suspected cause — UNVERIFIED, not traced to source

> **Superseded by encounter `#3`.** Traced to `BaseScalability.ini:178`: the group is
> **`ShadowQuality`**, not `EffectsQuality`. The paragraph below is kept as the record of what
> was believed; do not act on it.

`r.VolumetricFog` is believed to be driven by the **effects scalability group**, with
`sg.EffectsQuality` at Low/Medium setting it to 0. This has **not** been confirmed against
`Scalability.ini` or against the cvar's `ECVF_Scalability` flag in this investigation — an
editor running at reduced effects quality merely *appeared* to be the condition under which
the cvar was found at 0. Do not write the scalability chain into a fix without checking it.

What IS established, and is enough for the fix: the cvar was `0`, the verb answered
`enabled: true`, and setting the cvar to `1` is what made the pass render.

## Why the fix is cheap: the plugin already reads cvars

`system.console.search` already returns a live `currentValue` per row straight out of the
`IConsoleManager` registry — confirmed this session:

```js
call("system.console.search", {query: "r.VolumetricFog", limit: 4,
                               fields: ["name", "currentValue"]})
// -> {"results":[{"name":"r.VolumetricFog","currentValue":"1"}, ...]}
```

So `lighting.setup_volumetric_fog` needs no new machinery to measure the thing that vetoes
it — only an `IConsoleManager::Get().FindConsoleVariable(TEXT("r.VolumetricFog"))` read at
the same point it already writes the component flag.

## Suggested fix

`enabled` should mean "volumetric fog will render", not "the component flag is set" — the
same measured-not-requested house style the capture verbs already use for their `viewport.*`
blocks.

- **Minimum**: report the measured cvar, and emit a `cvarWarning` naming `r.VolumetricFog`
  and its value when the component flag is set and the cvar is 0.
- **Better**: the verb sets the cvar itself and reports both requested and measured. It
  already exists to spare the caller a multi-call dance, exactly as the `post_process.*`
  setters flip `bOverride_*` on the caller's behalf.

The cvar is live-only and dies with the editor, so a fix that sets it should say so; durable
pinning is a `[SystemSettings]` line in the project's `DefaultEngine.ini`.

## Generalise: every verb a scalability cvar can veto

This is one instance of a class. The same measure-and-report treatment is worth considering
for `lighting.setup_global_illumination`, `post_process.set_lumen_gi`,
`post_process.set_lumen_reflections` and `lighting.configure_shadows` — each writes state
whose effect a cvar can silently nullify. Related in shape: `B-set-scalability-no-sg-update`,
`B-lumen-update-scene-silent-success`.

## Related

- `B-set-component-properties-no-change-notification` (IN-REVIEW) — stacks with this one.
  A fog edit had **two** independent silent-failure paths in front of it: the write might not
  notify, and the pass might not be running. Roughly fifteen capture cycles went into
  separating them, and one fully wrong defect report was filed and retracted in between.
- `B-property-set-container-empty-change-event` (OPEN) — the other half of that stack.

## History
- `#1-cvar-veto-found-on-atlantis` `OPEN` reporter — Found while building the Atlantis example level on host project EAContentExamples58. `lighting.setup_volumetric_fog {viewDistance: 65000}` answered `{success:true, enabled:true}` and `property.get bEnableVolumetricFog` answered `true` while no volumetric fog was in any frame; `r.VolumetricFog` was 0. Toggling the cvar off moved the frame 3e-4 (it was already off); toggling it on took mean luminance 0.194400 -> 0.383695 at a fixed pose with pinned exposure, and produced the level's first god-ray shaft. Every fog value on that level then had to be re-tuned, because all of them had been judged against a frame with no volumetric fog in it (`FogDensity` 0.2 -> 0.06, `VolumetricFogAlbedo` (130,195,225) -> (60,130,160)). Worked around by pinning `r.VolumetricFog=1` under `[SystemSettings]` in the project's `DefaultEngine.ini`; re-confirmed still `1` after a machine restart this session via `system.console.search`. The scalability-group cause is recorded as an explicitly unverified hypothesis — it was never traced to `Scalability.ini` or to the cvar's flags. The plugin defect is untouched: the verb still answers `enabled: true` when volumetric fog cannot render.
- `#2-second-confirmation-and-cvar-grep` `OPEN` reporter — Second independent confirmation, same session and same host project (EAContentExamples58), reaching the same defect by the same route; the repro, the three-row luminance table, the captures, the god-ray acceptance test and the ini pin are already in this ticket and are not restated. Two things this ticket did not have. (1) **The spec this makes look impossible.** `Docs/map/atlantis-spec.md:114` § *Continuous motion (no gameplay code, no Blueprints)* specifies "**God rays** — volumetric fog + directional light volumetric scattering, not geometry cards". With the pass merely switched off and the verb answering `enabled: true`, that requirement reads as unachievable in the engine rather than as one cvar away — which is how it was read for most of a session before the cvar was found. A verb that reports the veto turns a spec-level dead end back into a one-line fix. (2) **Verified source fact narrowing the fix site.** `grep -rn "r\.VolumetricFog"` over `Plugins/PinWright/Source`, `Plugins/PinWright/docs` and `Plugins/PinWright/Content` returns **zero** hits at this checkout's HEAD — the string exists nowhere in plugin code, docs or content, so the verb has no read of the cvar available to report. (Qualification, stated because the grep is only clean when scoped: the string does occur in captured engine startup logs under `Plugins/PinWright/dist/*.log`, e.g. `load-smoke.log:654` `Set CVar [[r.VolumetricFog:1]]` followed by `:852` `Set CVar [[r.VolumetricFog:0]]` — engine output, not plugin source, though that off-after-on pair is consistent with this ticket's unverified scalability-group hypothesis and is a cheap place for a fixer to start.) Meanwhile the same handler already touches render state one line below where the read belongs: `PinWright::MarkComponentRenderStateDirty(FogComp)` at `Handlers/Environment/LightingHandler.cpp:698`, immediately after `bEnableVolumetricFog` and `VolumetricFogDistance` are written at `:690`/`:694` and immediately before the response is assembled at `:700-706`. So the fix is a single missing `IConsoleManager` read at a point the handler is already open, feeding the `enabled` field the response already sets at `:703`. Status and severity unchanged.
- `#3-scalability-chain-traced-and-a-second-veto-cvar-found` `OPEN` reporter — Same session and host project (EAContentExamples58), reached while making the Atlantis god rays render. Two additions; the repro, luminance table and ini pin already in this ticket are not restated. **(1) The "Suspected cause — UNVERIFIED" section can now be closed, and its hypothesis was wrong about which group.** `r.VolumetricFog=0` is set by **`[ShadowQuality@1]`**, not by the effects group: `C:/UE_5.8/Engine/Config/BaseScalability.ini:178`, inside the block that starts at `:167`. `[EffectsQuality@1]` (`:827-854`) does not mention `r.VolumetricFog` at all, and `[ShadowQuality@3]` sets it back to `1` at `:251`. So the veto follows **shadow** quality, and a fixer chasing `sg.EffectsQuality` would have found nothing. Measured live on this editor: every scalability group reads `1` (Low) via `system.console.search {query:"sg."}`, which is exactly the state that zeroes the cvar, and `r.VolumetricFog` reads `1` only because the `[SystemSettings]` pin in `Config/DefaultEngine.ini` overrides it. The pin is load-bearing, not belt-and-braces. **(2) A second cvar with the identical shape, on the same light, found the same way.** `r.LightShaftQuality` is set to `0` by `[PostProcessQuality@1]` (`BaseScalability.ini:602`). While it is `0`, the directional light's `bEnableLightShaftBloom` and `bEnableLightShaftOcclusion` are decorative in exactly the way `bEnableVolumetricFog` is: `actor.describe` on `Sun_Filtered` reported both as `true` and no light shaft was rendered. Setting `r.LightShaftQuality 1` at a fixed pose with pinned exposure took the frame from a featureless gradient to a dozen discrete radiating shafts and moved mean luminance 0.616936 -> 0.443637 (it *darkens*, because `bEnableLightShaftOcclusion` is what supplies the contrast between shafts). Captures `godray_t5_res512_sp100_B.png` and `godray_t6_lightshafts_B.png` under `Saved/Screenshots/OpenLevel/`. Worked around the same way, a `[SystemSettings]` line in `Config/DefaultEngine.ini` beside the existing `r.VolumetricFog=1`. This strengthens the "Generalise" section above with a confirmed second member rather than a suspected one: the audit should cover any component flag whose renderer pass a scalability cvar can switch off, and `bEnableLightShaftBloom` / `bEnableLightShaftOcclusion` on `ULightComponent` are now known members. Status and severity unchanged.
- `#4-enabled-measured-against-r-volumetricfog` `IN-REVIEW` developer — `lighting.setup_volumetric_fog` now reads `r.VolumetricFog` through `IConsoleManager` at the point it already marks render state dirty, and `enabled` is measured as (component flag AND cvar non-zero) instead of the literal `true` it used to be. The written flag is reported separately as `componentFlag`, the measurement as `volumetricFogCVar {cvar, found, value}` (`value` absent when the registry does not carry the cvar), and a vetoed call carries `cvarWarning` naming the inert properties, the `system.console_command "r.VolumetricFog 1"` session fix, the `[SystemSettings]` ini pin and the `ShadowQuality@1` scalability source from encounter `#3`. The registration summary states the measured contract. The verb deliberately does NOT set the cvar (the ticket's "Better" option): flipping a scalability cvar permanently is global state a verb must not change silently, and reporting is the treatment that generalises to the sibling verbs listed under "Generalise". Files: `Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp` (the `setup_volumetric_fog` handler only), new `Source/PinWright/Private/Tests/Environment/TestVolumetricFogCVarVeto.cpp`. Test `PinWright.lighting.setup_volumetric_fog.EnabledFollowsCVarVeto` — differential: same fixture, same call, run at `r.VolumetricFog` 0 and 1, asserting `enabled` false/true respectively, `componentFlag` true in both, `cvarWarning` present only when vetoed, and that the two runs disagree. The old hardcoded `true` fails the 0 run outright. Not compiled or run here — the orchestrator owns builds.
- `#5-verified-fixed-behaviourally` `DONE` verifier — 2026-08-28. Differential repro on the rebuilt binary at HEAD `b79ba53e` against the live editor on `/Game/Maps/Atlantis`: the SAME call, `lighting.setup_volumetric_fog {}`, run either side of an `r.VolumetricFog` toggle, against the level's existing `ExponentialHeightFog_0` whose `bEnableVolumetricFog` was already `true`, so nothing on the level was changed and nothing was saved. **At `r.VolumetricFog 1`** → `{enabled:true, componentFlag:true, volumetricFogCVar:{cvar:"r.VolumetricFog", found:true, value:1}}`, no `cvarWarning`. **At `r.VolumetricFog 0`** → `{enabled:false, componentFlag:true, volumetricFogCVar:{… value:0}}` plus `cvarWarning` naming every inert property (`VolumetricFogAlbedo`, `VolumetricFogEmissive`, `VolumetricFogDistance`, `VolumetricFogExtinctionScale`, `VolumetricFogScatteringDistribution`, every light's `VolumetricScatteringIntensity`), the session fix, the `[SystemSettings]` ini pin and the `ShadowQuality@1` scalability source from encounter `#3`. The two runs disagree on `enabled` while `componentFlag` reads `true` in both — the old hardcoded `true` cannot produce that. **`enabled:false` proven truthful rather than merely different, by capture and not by reading the value back:** identical pose `{x:-10500,y:-3800,z:1500}` / `{pitch:4,yaw:40,roll:0}` / `fov 70`, 448x448, game view on, `hideEditorSprites:true`, pinned `exposure {mode:"fixed", ev100:0}`, `r.ScreenPercentage 100` — `verify_vfog_off.png` `meanLuminance` 0.167334 against `verify_vfog_on.png` 0.272931, and the images differ exactly as the ticket describes: the cvar-0 frame is dark and hard-edged with no volumetric haze, the cvar-1 frame carries the volumetric wash filling the space between the columns. So the verb reports `enabled:false` precisely in the state where the renderer draws no fog. `r.VolumetricFog` was restored to `1` (the `[SystemSettings]` pin's value) and the restoration confirmed by the second capture rendering fog again. The verb correctly does NOT set the cvar.
