---
id: B-capture-verbs-silent-default-material-fallback
title: "asset.generate_thumbnail and render.capture_asset_preview render the engine Default Material with success:true when the subject's master material failed to compile — two different material instances returned byte-identical imageStats and neither response says the material fell back"
status: OPEN
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

## History
- `#1-filed` `OPEN` reporter — Hit while authoring the FPS environment master material on UE 5.8 / EAContentExamples58. Sequence: `material.compile_mgir` (`blocksCompiled:1, expressionsCreated:50`), `asset.save` (`saved:true, sizeBytes:45205`), then `asset.generate_thumbnail` x3 — once on the master and once on each of two very different instances — all three returning `success:true` with the statistics quoted above, byte-identical between the two instances. `render.capture_asset_preview` on a mesh using the same master returned a near-black default-material frame whose `rangeWarning` blamed exposure. The cause was a single sampler-type mismatch on `T_TilingNoise_001` (Color where the sampler wanted Linear Color), reported correctly and immediately by `material.authoring.compile_material` — so the information exists in the engine at capture time and the capture verbs simply do not ask for it. Fixed on my side by declaring `SamplerType: "SAMPLERTYPE_LinearColor"` on that texture-object parameter; the same defect then reappeared on `M_ENV_Decal_TireMark` (a Normal-map texture sampled as Color), again invisible to the capture path. Dedup: grepped the board for `generate_thumbnail`, `shader map`, `shaderMapComplete`, `compiledWithErrors` and `default material`. Distinct from `B-thumbnail-cold-first-frame-no-stats` (DONE — a transient cold *first* frame with no statistics published at all; here the statistics are published, look healthy, and the condition is permanent), from `B-compile-material-false-shader-success` (DONE — `compile_material`'s own reporting, which now works correctly and is what diagnosed this), and from `B-capture-asset-preview-renders-foliage-black` (a foliage-specific lighting/preview-scene case, not a material-compile fallback). Workaround in force for my stream: call `material.authoring.compile_material` and assert `compiledWithErrors:false` on every authored material before trusting any capture of anything that uses it.
