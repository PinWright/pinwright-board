---
id: B-capture-verbs-silent-default-material-fallback
title: "asset.generate_thumbnail and render.capture_asset_preview render the engine Default Material with success:true when the subject's master material failed to compile — two different material instances returned byte-identical imageStats and neither response says the material fell back"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, generate_thumbnail, render, capture_asset_preview, material, shader-map, default-material, false-evidence, silent-false-success, visual-verification, readiness]
encounters: 1
lastSeen: 2026-09-02T19:20:00+00:00
---

# A capture of a broken material looks like a capture of a material

## Symptom

`asset.generate_thumbnail` on `/Game/FPS/Env/Materials/M_ENV_Surface` (a freshly compiled master)
returned:

```jsonc
{ "success": true, "blank": false, "crushed": false, "blownOut": false,
  "imageStats": { "meanLuminance": 0.17077134235499042,
                  "luminanceVariance": 0.030324045654571474,
                  "toneLevelsUsed": 44, "litPixelFraction": 0.9824447631835938 },
  "readiness": { "assetCompilationWaited": true, "shaderMapChecked": true,
                 "shaderMapCompleteBefore": false, "shaderMapCompleteAfter": false } }
```

The PNG is the engine **Default Material** — a flat grey with its world-space checker grid. Read
with vision, it is a plausible-looking neutral surface, which is exactly why it passes.

The material had not "not finished compiling". It could **never** compile:

```
material.authoring.compile_material {assetPath: "/Game/FPS/Env/Materials/M_ENV_Surface"}
-> { "compileSucceeded": false, "compiledWithErrors": true, "compileStatus": "failed",
     "compileErrors": ["(Function WorldAlignedTexture) (Node TextureSample) Sampler type is
       Color, should be Linear Color for /Game/ExampleContent/Substrate/Textures/T_TilingNoise_001"] }
```

## The tell that should have been the response, and was not

Two **different** material instances of that broken master —
`MI_ENV_Asphalt_Wet` (BaseTint 0.185/0.185/0.195, Wetness 0.5) and
`MI_ENV_Container_Rust_Orange` (BaseTint 0.62/0.24/0.09, Metallic 0.72, different BaseTex and
NormalTex) — returned `imageStats` that are **equal to the last decimal place**, and equal to the
master's:

```
meanLuminance      0.1706355449306739   (both instances)   0.17077134235499042 (master)
luminanceVariance  0.03024591204555136  (both instances)
minLuminance       0.019607843137254898 maxLuminance 0.5218878431372549 (all three)
toneLevelsUsed     44                   litPixelCount 257525 (both instances)
```

Byte-identical statistics from two materials that share nothing but a parent is a categorical
signal that neither frame drew the material asked for. The verb computes those numbers and does
not compare them to anything, so the signal exists in the payload and is never raised.

`render.capture_asset_preview` is worse: on `/Game/FPS/Env/Meshes/SM_ENV_Container20` it produced
a near-black frame with the default material's grid, `blank: false`, `crushed: true` — and
`crushed` was attributed by `rangeWarning` to exposure ("LOWER `ev100`"), pointing the caller at
the wrong knob entirely. That verb publishes **no** `readiness` block at all, so it has not even
the weak signal the thumbnail path has.

## Why the existing signal does not close it

`readiness.shaderMapCompleteAfter: false` is present and is the only true thing in the response.
It is not enough, for three reasons:

1. It is a **readiness** field, so it reads as "still warming up" — the documented, transient,
   retry-and-it-goes-away condition of `B-thumbnail-cold-first-frame-no-stats` (DONE). Here it is
   permanent: the shader map will never complete, and a retry produces the identical frame
   forever. Nothing distinguishes "not yet" from "never".
2. It sits beside `success: true`, `blank: false`, `crushed: false` and a full `imageStats` block
   that all say the frame is good.
3. `render.capture_asset_preview` does not publish it at all.

## What should happen

The engine already knows: `UMaterial::GetMaterialResource(GMaxRHIFeatureLevel)->GetCompileErrors()`
is non-empty, and the primitive is rendering `UMaterial::GetDefaultMaterial`. Either fact is a
one-line probe at capture time. Any of these closes it:

- **Report the fallback.** Add `materialFallback: {occurred, subjects[], reason: "shaderMapFailed"}`
  (and the compile errors, which `MaterialCompileErrorCollector::WaitAndCollect` already collects
  for `compile_material` — see `B-compile-material-false-shader-success` `#2`) to both verbs, and
  raise it as a `warnings[]` entry so it survives a caller who only reads the top level.
- **Separate "not yet" from "never"** in `readiness`: `shaderMapFailed: true` when the resource
  carries compile errors, versus `shaderMapCompleteAfter: false` for a compile still in flight.
- **Publish `readiness` on `render.capture_asset_preview`** as well; it has the same exposure.
- Cheapest partial: when `imageStats` of a capture is bit-identical to the previous capture of a
  *different* asset in the same session, say so.

## Why it matters here

The project's binding rule is that a capture nobody looked at does not count, so an agent doing
the right thing — capture it, read the PNG, describe it — is precisely the one this defect
defeats. I looked at three PNGs and wrote down "grey cube with a faint checkerboard", which is a
truthful description of a frame that was evidence of nothing. The real defect was found only by
calling `material.authoring.compile_material` by hand on a hunch, which is a verb a caller has no
reason to run after `material.compile_mgir` has already returned `blocksCompiled: 1` and
`asset.save` has returned `saved: true, sizeBytes: 45205`.

severity rationale: impact=false visual evidence that survives the documented look-at-it check,
on the verb pair the whole visual-review workflow rests on x reach=every material authored through
MGIR or the graph verbs whose shader map fails, which is the normal outcome of a sampler-type or
HLSL mistake -> High

## Fix

Root cause: both capture handlers treated a non-empty render as sufficient evidence and never fed
the captured asset's materials through the existing shader-state probe. The thumbnail readiness
booleans could say a shader map was incomplete, but they did not distinguish a compile still in
flight from a permanent compile error; the asset-preview response had no material readiness at all.

Changed `MaterialShaderState.h` to provide one production capture-readiness serializer and fallback
policy over direct material, Static Mesh, and Skeletal Mesh subjects. It probes the actual material
interface resource, caches only interfaces that share that resource, and therefore preserves a
material instance's static-permutation errors. It reports null slots and identifies its mesh usage
policy explicitly: thumbnail capture uses LOD0 because both engine thumbnail renderers disable LOD
selection, while asset-preview capture reads the provider's actual preview component after the draw.
Skeletal previews use forced/predicted LOD; Static Mesh previews use forced LOD or an explicit LOD0
fallback because that component has no game-thread predicted-LOD API. Preview/hidden section filters
keep inactive slots out of refusal policy. Intentional use of the engine Default Material reports
`usingDefaultMaterial:true` without a false fallback, while debug modes that replace subject
materials do not claim the broken subject reached the captured pixels.

The final capture probe gives already-submitted shader jobs one bounded, game-thread-pumped wait.
Because UE's render proxy substitutes Default Material whenever its render-thread shader map is
incomplete, a still-`notCompiled`, `outstanding`, or `timedOut` used material is reported as
uncertain rather than promoted to a known failure: the capture succeeds without opt-in, publishes
`fallbackPossible:true` with `possibleReason:"shaderMapIncomplete"`, keeps the exact subject status,
and warns that the frame may use fallback. Only a measured compile failure with errors or an
unassigned rendered slot sets `fallbackOccurred:true` and fails closed.

`AssetWorkflowHandler.cpp` and `RenderHandler.cpp` share one `allowFallback` parameter definition
(default `false`). Thumbnail readiness is re-probed after the final render/retry, and thumbnail/file
generation failures keep `THUMBNAIL_GENERATION_FAILED` priority. A known substitution returns the
new `MATERIAL_FALLBACK` code from `ErrorCodes.h` with structured readiness/image details; explicit
opt-in retains the image and adds a reason-aware warning that points to compile errors or the
unassigned slot. Incomplete states add their own warning without requiring opt-in. Updated only the
changed contract pages: `asset.md`, `render.capture-subjects.md`, and
`material.compile-state.md`.

Files changed for this fix:

- `Source/PinWright/Private/Handlers/Material/MaterialShaderState.h`
- `Source/PinWright/Private/Handlers/ErrorCodes.h`
- `Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp`
- `Source/PinWright/Private/Handlers/Render/CaptureSubjectProviders_Mesh.h`
- `Source/PinWright/Private/Handlers/Render/CaptureSubjectProviders_Mesh.cpp`
- `Source/PinWright/Private/Handlers/Render/RenderHandler.cpp`
- `Source/PinWright/Private/Tests/Material/MaterialShaderStateTestFixtures.h`
- `Source/PinWright/Private/Tests/Material/TestCompileMaterialShaderErrors.cpp`
- `Source/PinWright/Private/Tests/Material/TestMaterialShaderStateReport.cpp`
- `Source/PinWright/Private/Tests/Render/TestAssetPreviewSubjects.cpp`
- `Source/PinWright/Private/Tests/Render/TestCaptureAssetPreviewMaterialFallback.cpp`
- `Docs/wiki-src/asset.md`
- `Docs/wiki-src/material.compile-state.md`
- `Docs/wiki-src/render.capture-subjects.md`

Regression tests:

- `PinWright.material.shader_state.CaptureReadinessStructure`
- `PinWright.asset.generate_thumbnail.MaterialFallbackRequiresOptIn`
- `PinWright.asset.generate_thumbnail.MaterialInstanceFallbackRequiresOptIn`
- `PinWright.render.capture_asset_preview.MaterialFallbackRequiresOptIn`

The shared deliberate invalid-HLSL fixture is also used by the existing compile-error handler test.
It feeds a real `ProbeAndWait` result through the production capture policy and uses
`PinWrightTestSkip` when shader compilation cannot produce the fixture. The structural test covers
successful warned uncertainty for all three incomplete states, captured-component LOD/section scope,
and known-fallback refusal; the
handler tests cover default rejection, actual material-instance permutation probing, explicit
opt-in, retained thumbnail output, and clean-material controls.

Deliberately unchanged: image heuristics and `imageStats`, cold-frame readiness/retry behavior,
capture primitives, open-level capture, and material discovery for animation/Niagara subjects that
do not directly expose mesh material slots on the loaded asset. There is no early fallback refusal:
rendering can finalize compilation, and the decision must describe final pixels and view-mode
material substitution rather than a stale preflight state.

## History
- `#1-filed` `OPEN` reporter — Hit while authoring the FPS environment master material on UE 5.8 / EAContentExamples58. Sequence: `material.compile_mgir` (`blocksCompiled:1, expressionsCreated:50`), `asset.save` (`saved:true, sizeBytes:45205`), then `asset.generate_thumbnail` x3 — once on the master and once on each of two very different instances — all three returning `success:true` with the statistics quoted above, byte-identical between the two instances. `render.capture_asset_preview` on a mesh using the same master returned a near-black default-material frame whose `rangeWarning` blamed exposure. The cause was a single sampler-type mismatch on `T_TilingNoise_001` (Color where the sampler wanted Linear Color), reported correctly and immediately by `material.authoring.compile_material` — so the information exists in the engine at capture time and the capture verbs simply do not ask for it. Fixed on my side by declaring `SamplerType: "SAMPLERTYPE_LinearColor"` on that texture-object parameter; the same defect then reappeared on `M_ENV_Decal_TireMark` (a Normal-map texture sampled as Color), again invisible to the capture path. Dedup: grepped the board for `generate_thumbnail`, `shader map`, `shaderMapComplete`, `compiledWithErrors` and `default material`. Distinct from `B-thumbnail-cold-first-frame-no-stats` (DONE — a transient cold *first* frame with no statistics published at all; here the statistics are published, look healthy, and the condition is permanent), from `B-compile-material-false-shader-success` (DONE — `compile_material`'s own reporting, which now works correctly and is what diagnosed this), and from `B-capture-asset-preview-renders-foliage-black` (a foliage-specific lighting/preview-scene case, not a material-compile fallback). Workaround in force for my stream: call `material.authoring.compile_material` and assert `compiledWithErrors:false` on every authored material before trusting any capture of anything that uses it.
- `#2-fail-closed-fallback` `IN-REVIEW` developer — Added shared material-readiness probing and serialization, `allowFallback:false` on both capture verbs, dedicated `MATERIAL_FALLBACK` refusal with structured details, opt-in warning behavior, and the three regression tests listed in Fix. Existing pixel statistics and cold-frame evidence remain unchanged.
