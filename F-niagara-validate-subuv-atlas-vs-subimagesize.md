---
id: F-niagara-validate-subuv-atlas-vs-subimagesize
title: "niagara.validate has no check that a sprite/mesh renderer's SubImageSize matches the assigned SubUV texture's actual atlas layout — a 6x6 flipbook on an 8x8 renderer validates clean at strict and renders black gutter"
status: OPEN
severity: High
category: feature
tags: [niagara, niagara-validate, subuv, subimagesize, sprite-renderer, flipbook, texture, silent-wrong-render, vfx]
encounters: 1
costly: 1
lastSeen: 2026-09-07
---

# A SubUV renderer whose frame grid does not match its texture is invisible to every verb

`UNiagaraSpriteRendererProperties::SubImageSize` declares the flipbook grid the
vertex factory slices the assigned texture into. Nothing in the plugin compares it
against the texture that is actually assigned, so a renderer set to `8 x 8` over a
`6 x 6` atlas is reported as healthy by every read and validation surface there is.

The failure is total and silent: `NiagaraSpriteVertexFactory.ush:1068` remaps
`TexCoord0` into `(SubImageCol + u, SubImageRow + v) * SubImageSize.zw`
unconditionally, so every sprite samples a `1/8 x 1/8` window of a texture whose
cells are `1/6 x 1/6`. The window straddles cell boundaries almost everywhere: the
sprite shows mostly inter-cell black with a clipped fragment of a puff shoved
against one edge, never a centred frame. The `SubUVAnimation` module compounds it,
because its default `Define SubUV Setup Manually: Automatic (From Renderer SubImage
Size)` takes the frame count from the same wrong property and steps
`NormalizedAge x 64` over a 36-cell atlas.

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-07

`/Game/FPS/VFX/NS_Explosion` emitter `Fireball`, renderer index 0:

| | value | source |
|---|---|---|
| `SubImageSize` | `{X: 8, Y: 8}` | `niagara.inspect` renderer properties |
| Material | `MI_FPS_Fire_ExplosionCore` -> `M_FPS_FireSubUV_Add` | same |
| Flipbook texture | `/Game/ExampleContent/Effects/Textures/T_Fire_subUV_01` | `asset.dump` `mgir.txt`, `MaterialExpressionParticleSubUV` |
| Texture size / format | 1024x1024, `PF_DXT1`, sRGB, grayscale, alpha constant 255 | `asset.dump` `texture.json`, `texture.get_pixel_stats` |
| **Actual atlas layout** | **6 x 6 (36 frames)** | measured: column/row luminance profiles both show 5 interior gutters at ~43 px spacing over 256 px |

`niagara.validate {level: "strict"}` on that system returns no issue of any kind
about the renderer. `niagara.inspect` reports `SubImageSize` and the material, and
nothing joins the two. This emitter has never produced a visible frame across four
review rounds; three separate diagnoses (unstable sort order, no-alpha flipbook,
under-driven colour) were filed against it and none of them was the cause.

The control that makes it conclusive is in the same system: `SmokeColumn` and
`DustRing` are also at `SubImageSize 8 x 8`, and both use
`T_SmokeSubUV_8X8` — a genuine 8x8 atlas — and both render correctly. One property,
one mismatch, one emitter that has never worked.

## What is wanted

A check on `niagara.validate` (both levels; this is not a `strict`-only concern —
it renders wrong under any reading of the asset) that, for every enabled renderer
with `SubImageSize` other than `(1,1)`, resolves the renderer's material to the
`MaterialExpressionParticleSubUV` / `MaterialExpressionTextureSample` texture it
samples and reports:

- `SUBUV_ATLAS_SIZE_INDETERMINATE` (warning) when the texture cannot be resolved or
  its layout cannot be measured — never silence, which reads as "checked and fine".
- `SUBUV_ATLAS_SIZE_MISMATCH` (error) when a layout *is* measurable and disagrees,
  naming the emitter, renderer index, declared `SubImageSize`, texture path, and
  measured grid.

Detecting the grid from pixels is the honest version (gutter periodicity in the
row/column luminance profiles resolves it in one pass over a mip) but a cheaper
first cut is worth shipping alone: **the texture dimensions must be divisible by
`SubImageSize` and the resulting cell must be square** for the overwhelming majority
of real atlases. 1024/8 = 128 passes that weak test, so it would not have caught
this one; a name-based heuristic (`T_SmokeSubUV_8X8`) would have, and is worth
emitting as a warning. The pixel measurement is the check that actually works.

## Related

- `F-texture-pixel-stats-region` — `texture.get_pixel_stats` is whole-mip only, so
  today there is no plugin surface that can measure an atlas grid at all; I had to
  hand-decode the editor thumbnail out of the `.uasset` bytes with a stdlib PNG
  decoder to establish the 6x6.
- The same 1/8 sub-tile remap silently breaks any material that builds a mask from a
  raw `TextureCoordinate` node on a SubUV sprite (`M_FPS_FireSubUV_Add`,
  `M_FPS_Dust_Lit` in this project). That is a content defect, not a plugin one, but
  a validate check that knows the SubUV grid is where it would be caught.

## History
- `#1-filed` `OPEN` reporter — Filed while diagnosing a four-round "the explosion has no fireball" defect on `/Game/FPS/VFX/NS_Explosion`. `SubImageSize (8,8)` on `Fireball`'s sprite renderer against a `T_Fire_subUV_01` that is measurably a 6x6 / 36-frame atlas; `niagara.validate {strict}` clean, `niagara.inspect` reports both facts without joining them, and the two sibling emitters at 8x8 over a genuine 8x8 texture render correctly. Grid measured from the texture's embedded 256x256 editor thumbnail (decoded out of the `.uasset` with stdlib zlib, since no plugin verb returns sub-region pixel content): column and row luminance profiles each show five interior gutters at ~43 px spacing, and an 8x8 overlay cuts through the middle of every puff while a 6x6 overlay lands in the gutters. Ripgrep over the board for `SubImageSize` / `subuv` / `subimage` / `flipbook` found no existing ticket.
