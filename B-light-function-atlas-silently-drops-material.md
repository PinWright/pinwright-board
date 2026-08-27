---
id: B-light-function-atlas-silently-drops-material
title: "A light function material is silently dropped from the light function atlas, so it modulates surfaces but NOT volumetric fog, and no PinWright read-back distinguishes that from a working setup"
status: OPEN
severity: High
category: bug
tags: [lighting, light-function, volumetric-fog, god-rays, material, atlas, silent-noop, measured-vs-requested, directional-light]
encounters: 1
lastSeen: 2026-08-27T19:40:00+05:00
---

# The light function paints the floor and does nothing to the fog, and everything reads healthy

A `Material` assigned as `LightFunctionMaterial` on a directional light drives **two**
independent renderer paths:

- the classic deferred light-function pass, which modulates **opaque surfaces**; and
- the **light function atlas** (`r.LightFunctionAtlas`), which is what volumetric fog samples
  (`r.VolumetricFog.UsesLightFunctionAtlas`, aliased as `r.VolumetricFog.LightFunction`).

The second path silently drops materials the first accepts. When it does, the light function
still paints beautiful animated caustics across the seafloor while contributing **nothing** to
the volumetric medium — which is exactly the "god rays from a light function" effect it was
attached for.

Every read-back available to a caller reports success:

```js
call("actor.get_component_property", {actorName:"DirectionalLight_0",
  componentName:"LightComponent0", propertyName:"VolumetricScatteringIntensity"})
// -> {"value": 4}

call("actor.describe", {actorName: "DirectionalLight_0"})
// -> LightFunctionMaterial: "/Game/Atlantis/Materials/M_LF_Caustics.M_LF_Caustics"
//    LightFunctionScale: [3000,3000,3000]  LightFunctionFadeDistance: 200000
```

and every relevant cvar reads enabled:

```js
call("system.console.search", {query:"LightFunction", kind:"variable",
                               fields:["name","currentValue"]})
// r.LightFunctionAtlas                    1
// r.VolumetricFog.UsesLightFunctionAtlas  1
// r.VolumetricFog.LightFunction           1
// r.LightFunctionAtlas.MaxLightCount     -1
```

All green. The fog still has no light function in it.

## Root cause, traced to engine source

`Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp:1769`:

```cpp
MaterialCompilationOutput.bIsLightFunctionAtlasCompatible =
    (!bUsesVertexPosition && !bUsesSceneDepth && !MaterialCompilationOutput.bNeedsSceneTextures
     && !bPotentiallyManipulateTexCoords)
    || Material->GetForceCompatibleWithLightFunctionAtlas();
```

The comment above it (`:1761-1767`) gives the reasoning: an atlas tile is rendered without
world-position or depth inputs, and scaled texcoords no longer align with the tile edges.

So **any animated or panning light function is excluded by construction** — scrolling or
scaling UVs sets `bPotentiallyManipulateTexCoords`, and a world-space-projected caustic sets
`bUsesVertexPosition`. The atlas builder then files it under
`NonCompatibleLightFunctionMaterials` (`Renderer/Private/LightFunctionAtlas.cpp:481-487`) and
renders nothing for it.

The escape hatch is a per-material checkbox, `UMaterial::bForceCompatibleWithLightFunctionAtlas`
(`Engine/Public/Materials/Material.h:926-927`, display name **"Compatible With Light Function
Atlas"**, category `LightFunctionMaterial`).

## The only existing diagnostic is a show flag, and it is not in the plugin

`ShowFlag.VisualizeLightFunctionAtlas 1` overlays the atlas and a legend that splits materials
into "Light Functions in atlas: N" and "Light functions not compatible with the atlas: N"
(`LightFunctionAtlas.cpp:913`, `:990`, `:1005`). That overlay is the single thing in the engine
that answers the question, and reaching it needs a raw `system.console_command` plus a capture.

Worse for a capture-driven workflow: **the legend is drawn at a fixed pixel offset and runs off
the right edge of a 768x768 frame**, so the counts are clipped and only the colour swatches and
the leading words survive. The class of the failure is legible; the numbers are not.

## Repro (host project EAContentExamples58, `/Game/Maps/Atlantis`)

Fixed pose, pinned exposure, game view on, `hideEditorSprites: true`, 768x768, camera
`location {x:-9000,y:0,z:400}` `rotation {pitch:45,yaw:215,roll:0}` `fov 60` — looking up the
water column toward the sun, where a shaft has to appear if it appears anywhere.

```js
// 1. baseline: light function attached, working on surfaces
call("render.capture_open_level", {/* pose above */})     // meanLuminance 0.611662

// 2. switch the fog's atlas sampling OFF - if the LF reached the fog, this must change it
call("system.console_command", {command:"r.VolumetricFog.UsesLightFunctionAtlas 0"})
call("render.capture_open_level", {/* same pose */})      // meanLuminance 0.609885

// 3. the fix: mark the material atlas-compatible
call("system.console_command", {command:"r.VolumetricFog.UsesLightFunctionAtlas 1"})
call("property.set", {objectPath:"/Game/Atlantis/Materials/M_LF_Caustics.M_LF_Caustics",
                      propertyName:"bForceCompatibleWithLightFunctionAtlas", value:true})
call("asset.save", {assetPath:"/Game/Atlantis/Materials/M_LF_Caustics"})
call("render.capture_open_level", {/* same pose */})      // shafts appear
```

## Evidence

| state | frame `meanLuminance` | `maxLuminance` | shafts in frame |
|---|---|---|---|
| light function attached, atlas sampling **on** | 0.611662 | 0.896902 | none — smooth gradient |
| atlas sampling forced **off** | 0.609885 | 0.896068 | none |
| `bForceCompatibleWithLightFunctionAtlas` **true** | 0.619473 | 0.912799 | radial streaks appear |

Disabling the fog's atlas sampling entirely moved the frame by **0.0018 — 0.29%**, against a
no-change control floor of 0.0017 measured on this same level. That is the whole finding in one
number: **the light function was contributing nothing to the fog, so removing it cost nothing.**

Atlas visualiser, same pose, before and after the checkbox:

- `godray_t2_atlasviz.png` — atlas entirely flat mid-grey, every slot empty; the material's red
  swatch sits under the "not compatible" heading.
- `godray_t3_atlasviz_after.png` — slot 0 allocated to the material, "not compatible" list empty.

Captures under `Saved/Screenshots/OpenLevel/` on EAContentExamples58:
`godray_base_B_upsun.png`, `godray_t1_noLFatlas_B.png`, `godray_t2_atlasviz.png`,
`godray_t3_atlasviz_after.png`, `godray_t4_lfatlas_B.png`.

**The surface caustics are unaffected by the fix.** Deferred directional lights keep using the
classic light-function pass, so the seafloor pattern is pixel-for-pixel the look it had before
(`godray_base_A_descent.png` mean 0.223353 vs `godray_t4_lfatlas_A.png` mean 0.225320, same
pattern and scale). There is no trade-off to weigh here, which is worth saying because a caller
staring at this would reasonably fear one.

## Suggested fix

The material's atlas compatibility is already a `FMaterialRelevance` bit
(`MaterialInterface.cpp:765`, `bIsLightFunctionAtlasCompatible`) reachable game-side via
`FMaterial::MaterialIsLightFunctionAtlasCompatible_GameThread()`. No new machinery is needed.

- **Minimum**: whenever a verb reports a light's `LightFunctionMaterial`
  (`actor.describe`, `actor.get_component_property`, `lighting.*`), report the measured
  compatibility bit beside it, and emit a warning naming
  `bForceCompatibleWithLightFunctionAtlas` when a light function is attached, volumetric fog is
  on, and the bit is false. That is the state where the caller believes they have god rays and
  does not.
- **Better**: a `lighting.diagnose_light_function` verb that answers the four-part question in
  one call — material assigned, atlas generation on, fog sampling the atlas, material accepted
  into it — and names which of the four is the veto.

Same measured-not-requested house style as the capture verbs' `viewport.*` blocks.

## Generalise

This is a second instance of the pattern in `B-setup-volumetric-fog-enabled-true-while-cvar-off`
and shares its shape exactly: **a lighting feature with more than one gate, where PinWright
reports the gate the caller set and stays silent about the gate that actually vetoes.** The fog
ticket's veto is a scalability cvar; this one's is a material shader-compilation bit. A caller
hitting either sees a healthy read-back and concludes the *engine* cannot do the thing.

Worth auditing the same way: `r.Translucent.UsesLightFunctionAtlas` and
`r.SingleLayerWater.UsesLightFunctionAtlas` are both **0** by default on this build, so a light
function silently does nothing on translucent and single-layer-water surfaces too, with no
diagnostic anywhere.

## Related

- `B-setup-volumetric-fog-enabled-true-while-cvar-off` (OPEN) — the sibling case, same level,
  same effect, different veto. Read them together: the Atlantis god rays needed BOTH fixed, and
  each one alone leaves the frame looking identically broken.

## History
- `#1-light-function-never-reached-the-fog` `OPEN` reporter — Found while making the Atlantis
  example level's god-ray shafts render on host project EAContentExamples58. `M_LF_Caustics` had
  been attached to `Sun_Filtered` with `LightFunctionScale 3000` and was demonstrably animating
  the seafloor; the volumetric fog it was supposed to band was a featureless gradient. Toggling
  `r.VolumetricFog.UsesLightFunctionAtlas` off changed the frame by 0.29% — at the level's
  measured no-change control floor — which proved the light function had never been in the fog
  at all. `ShowFlag.VisualizeLightFunctionAtlas 1` showed an entirely empty atlas with the
  material listed as not compatible; the rule was then traced to
  `HLSLMaterialTranslator.cpp:1769`, where any texcoord manipulation or vertex-position use
  excludes a material by construction unless
  `UMaterial::bForceCompatibleWithLightFunctionAtlas` overrides it. Setting that property and
  saving the asset put the material in the atlas and put radial shafts in the fog for the first
  time. Verified against disk per the host project's rule: `M_LF_Caustics.uasset` mtime
  14:21:13 -> 19:02:33, size 20396 -> 21250 bytes, and the string
  `bForceCompatibleWithLightFunctionAtlas` present in the file bytes afterwards and absent
  before (a default-valued bool is not serialized). The plugin defect is untouched: no verb
  reports the compatibility bit, and the only diagnostic in the engine is a show-flag overlay
  whose legend is clipped off the right edge of a square capture.
