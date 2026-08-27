---
id: B-setup-volumetric-fog-enabled-true-while-cvar-off
title: "lighting.setup_volumetric_fog reports enabled:true when r.VolumetricFog is 0, so the pass cannot render and every fog value tuned against that frame is tuned against the wrong renderer"
status: OPEN
severity: High
category: bug
tags: [lighting, setup-volumetric-fog, exponential-height-fog, cvar, scalability, silent-noop, misleading-success, measured-vs-requested]
encounters: 1
lastSeen: 2026-08-27T13:40:00+0000
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
