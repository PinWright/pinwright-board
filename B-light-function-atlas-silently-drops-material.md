---
id: B-light-function-atlas-silently-drops-material
title: "A light function material is silently dropped from the light function atlas, so it modulates surfaces but NOT volumetric fog, and no PinWright read-back distinguishes that from a working setup"
status: DONE
severity: High
category: bug
tags: [lighting, light-function, volumetric-fog, god-rays, material, atlas, silent-noop, measured-vs-requested, directional-light]
encounters: 2
lastSeen: 2026-08-28T09:20:00+05:00
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
- `#2-report-measured-atlas-compatibility` `IN-REVIEW` developer — `get_material_info` now emits a
  measured `lightFunctionAtlas` block for any MD_LightFunction material: `compatible` read off the
  compiled shader map via `FMaterial::MaterialIsLightFunctionAtlasCompatible_GameThread()` (omitted,
  not defaulted false, when there is no game-thread shader map), `forceCompatible` echoing
  `bForceCompatibleWithLightFunctionAtlas`, a measured `atlasGeneration` block for the
  `r.LightFunctionAtlas` cvar, and `warning` / `atlasWarning` naming the remedy for each gate. The
  incompatible warning states the translator rule and names the new setter. Added
  `material.authoring.set_light_function_atlas_compatible` (write + recompile + measured read-back,
  plus a `domainWarning` when the material is not a light function, since the flag is inert there),
  and `set_material_domain` now returns the same block when it switches a material INTO
  LightFunction. Logic lives in the new
  `Source/PinWright/Private/Handlers/Material/MaterialLightFunctionAtlas.h`; call sites in
  `Handlers/Material/MaterialAuthoringHandler.cpp`. Tests:
  `PinWright.material.authoring.get_material_info.LightFunctionAtlasCompatibility` and
  `PinWright.material.authoring.set_light_function_atlas_compatible.OverrideFlipsTheMeasuredBit`
  (`Tests/Material/TestMaterialLightFunctionAtlasCompatibility.cpp`). `MaterialDiscoveryHandler.cpp`
  was left unchanged: it catalogs expression CLASSES, and `bPotentiallyManipulateTexCoords` is a
  translator-time fact that cannot be derived from an expression CDO. Not addressed: `actor.describe`
  / `lighting.*` still report `LightFunctionMaterial` with no compatibility beside it, and the
  per-consumer `r.VolumetricFog.UsesLightFunctionAtlas` / `r.Translucent.*` / `r.SingleLayerWater.*`
  sampling switches are named in the warning text but not measured.
- `#3-verified-fixed-behaviourally` `DONE` verifier — 2026-08-28. Verified on the rebuilt binary at HEAD `b79ba53e` against the live editor, as a two-direction differential over a purpose-built scratch material rather than a read of the shipping one, so a `compatible:true` cannot be a field that is simply always true. Scratch material `/Game/PinWrightScratch/M_PwLFAtlasProbe`, built through the plugin's own verbs. **(1) The block appears on the domain switch.** `material.authoring.set_material_domain {materialDomain:"LightFunction"}` on the empty material returned `lightFunctionAtlas {forceCompatible:false, shaderMapReady:true, compatible:true, atlasGeneration:{cvar:"r.LightFunctionAtlas", found:true, value:1}}` — compatible, correctly, because an empty graph manipulates nothing. **(2) The measured bit really is measured.** Adding a `WorldPosition` node wired to `EmissiveColor` and calling `compile_material` (`compileSucceeded:true`) flipped `get_material_info` to `compatible:false` with a `warning` that states the translator rule verbatim (any TextureCoordinate node at all, a texture sample with `Coordinates` wired so any Panner / Rotator / scaled-UV chain reaches it, or a read of world position / scene depth / scene textures), names all three consumers it silently drops out of (`r.VolumetricFog.UsesLightFunctionAtlas`, `r.Translucent.UsesLightFunctionAtlas`, `r.SingleLayerWater.UsesLightFunctionAtlas`), explains why every other read-back looks healthy (the material keeps modulating opaque surfaces through the deferred pass), and names the new setter. Same asset, one graph edit apart, opposite answers — so the bit is read off the compiled shader map, not defaulted. **(3) The setter is a real write, not a report.** `material.authoring.set_light_function_atlas_compatible {forceCompatible:true}` returned `{forceCompatible:true, compatible:true}` — the measured bit flipped back on a graph that is still incompatible by construction, which is exactly what the override is for — and the write was verified against DISK per the host project's rule, not against the in-memory object: `Content/PinWrightScratch/M_PwLFAtlasProbe.uasset` mtime 2026-08-28 09:13:38, and `grep -a bForceCompatibleWithLightFunctionAtlas` on the file bytes hits (a default-valued bool is not serialized, so its presence in the bytes IS the write). This is the same evidence shape the reporter used on `M_LF_Caustics`. **(4) Both remaining gates warn.** A non-light-function target (`/Game/PinWrightScratch/M_PwLFAtlasSurfaceProbe`, domain Surface) returns `domainWarning` stating the flag can have no effect and naming `set_material_domain` as the prerequisite; and at `r.LightFunctionAtlas 0` the block gains `atlasWarning` saying the atlas is not generated at all so NO light function reaches any consumer regardless of this material's compatibility, with the session and `[SystemSettings]` remedies and the per-consumer cvars listed as separate switches. The cvar was restored to `1`. **Independent corroboration that the reported situation is genuinely resolved on this level:** while verifying the sibling ticket `B-showflag-cvar-override-contaminates-capture`, `ShowFlag.VisualizeLightFunctionAtlas 1` was forced and the resulting capture (`Saved/Screenshots/OpenLevel/verify_showflag_forced.png`) shows the atlas holding a populated caustic slot — where encounter `#1` saw "an entirely empty atlas with the material listed as not compatible". Not verified here and still open as the developer entry states: `actor.describe` / `lighting.*` report `LightFunctionMaterial` with no compatibility beside it, and the per-consumer sampling cvars are named in the warning text but not measured. Two scratch probe materials were left under `/Game/PinWrightScratch/` alongside the folder's existing probes; no shipping asset was touched.
