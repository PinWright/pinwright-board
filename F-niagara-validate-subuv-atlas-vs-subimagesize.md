---
id: F-niagara-validate-subuv-atlas-vs-subimagesize
title: "niagara.validate has no check that a sprite/mesh renderer's SubImageSize matches the assigned SubUV texture's actual atlas layout — a 6x6 flipbook on an 8x8 renderer validates clean at strict and renders black gutter"
status: IN-REVIEW
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
- `#2-subuv-atlas-check-shipped` `IN-REVIEW` developer — `niagara.validate` now checks every **enabled** renderer of every **enabled** emitter handle whose `SubImageSize` is other than `(1,1)`, and publishes `subUVAtlasCheck` (`not_applicable` / `consistent` / `mismatched` / `unverified`) on Niagara System results. New `Source/PinWright/Private/Handlers/Niagara/NiagaraSubUVAtlasCheck.h/.cpp` (namespace `PinWrightNiagara`, the file shape `NiagaraDataInterfaceConsistency` and `NiagaraCompileVerdict` already use) resolves the sampled texture in order — the renderer's own `MaterialParameters.TextureParameters` when there is exactly one (a per-instance MID overrides the material asset), else the renderer's sole material walked (including material functions, visited-set bounded) for a `ParticleSubUV` / `TextureSampleParameterSubUV` node, else the material's sole sampled texture; a renderer binding on the SubUV parameter's name or a material-instance override is resolved before measuring, and anything ambiguous is `unverified` rather than a guess, because a wrong texture yields a confident finding about the wrong asset. Three signals: divisibility (warning), an `NxM` spelled in the texture's asset name (warning; last run only, 1-2 digits each in 1-64, not part of a longer number, so `1024x1024` reads as a pixel size), and the measured one (error) — the atlas is read through the new `TexturePixelStats::MeasureTileGrid` at the *declared* grid and its wholly-empty trailing columns and rows are dropped, with a cell counted empty only at peak channel at or below 2/255, or fully transparent. `Source/PinWright/Private/Handlers/Niagara/NiagaraInspectHandler.cpp` gained `AddSubUVAtlasIssues`, called from the system branch beside the existing `Add*Issues` helpers and spelling its codes as raw literals like its siblings (adopting `ErrorCodes::` there would make the whole file registry-adopting and fail `PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly` on its existing hand-spelled codes). `NIAGARA_SUBUV_ATLAS_SIZE_MISMATCH` / `NIAGARA_SUBUV_ATLAS_SIZE_INDETERMINATE` registered in `Source/PinWright/Private/Handlers/ErrorCodes.h`; the reporter's spellings kept behind the `NIAGARA_` prefix the sibling IN-REVIEW validate tickets use. Severity tracks the strength of the evidence — measured is an error, a heuristic is a warning about the same fault — and both are reported at **both levels**, not as a strict-only escalation. Wiki: `Docs/wiki-src/niagara.md` gained `#### Does a SubUV renderer's frame grid match its atlas?` under `### niagara.validate`. Regression test `PinWright.niagara.validate.SubUVAtlasMismatch` (`Source/PinWright/Private/Tests/Niagara/TestNiagaraSubUVAtlasCheck.cpp`) drives a `UNiagaraSpriteRendererProperties` fixture at `SubImageSize (8,8)` bound to a 24x24 atlas populated only in the first 6 of 8 cells per axis. Counterfactual: with the rule absent the test does not compile; with the rule present but its measured half removed, `PopulatedGrid` stays (0,0), `IsMismatch()` is false and the mismatched renderer produces no finding at all. The test also pins a fully-populated atlas on the same declared grid raising nothing (no false positive), a `(1,1)` renderer being out of scope, an unresolvable texture producing a reason rather than silence, and the name parser on `T_Smoke_8x8` / `T_Fire_subUV_6X6_02` / `T_Fire_subUV_01` / `T_Sky_1024x1024`. **Known limitation, stated in the wiki rather than hidden:** a *fully packed* atlas at a different grid — this ticket's own `T_Fire_subUV_01`, 36 drawn cells declared as 8x8 — has no empty trailing cells, so only the name and divisibility heuristics can see it, and neither fires on that asset. Establishing a packed grid needs gutter periodicity in the row/column luminance profiles, which this check does not do; `consistent` therefore means "nothing measurable disagrees", not "the grid was proven correct". The pixel-measurement dependency `F-texture-pixel-stats-region` shipped alongside this and is what the measured signal reads through.
- `#3-gutter-period-detection-added` `IN-REVIEW` developer — Verifier follow-up on `#2`; three findings, all fixed. **(1) The check could not see this ticket's own asset.** Dropping wholly-empty trailing rows/columns keeps the cell size fixed, so on a fully packed sheet at the wrong grid — `T_Fire_subUV_01`, 36 drawn cells declared 8x8 — every declared tile overlaps a puff, `PopulatedGrid` reads (8,8), divisibility is quiet (1024/8 = 128) and the name spells nothing, so `niagara.validate` still answered `consistent`. Implemented the mechanism this ticket names: `PinWrightNiagara::DetectAxisDivisions` reads the cell period off a 1-D energy profile (one entry per texel column, and one per texel row, built with two `TexturePixelStats::MeasureTileGrid` strip passes; energy is mean luminance scaled by mean alpha so an additive sheet and an alpha-masked one both read). Each candidate division is scored by the BRIGHTEST sample landing on any of its interior cut lines — a real grid puts every cut in a gutter, a wrong one drives at least one through a frame — and the LARGEST division whose cuts all sit at or below 10 % of the profile mean wins, because a divisor of the true grid also lands its cuts in real gutters and the smallest passing division would read 2 for every atlas. The finding now carries `DetectedGrid` plus a per-axis confidence (1 - brightest-cut / profile mean) and a `DetectionReason` when no period is readable; the issue payload carries `detectedColumns` / `detectedRows`. **(2) Severity was over-reporting.** `NIAGARA_SUBUV_ATLAS_SIZE_MISMATCH` is now an **error** only for a measured cell-period disagreement; the trailing-empty, name and divisibility signals raise it as a **warning**. Trailing-empty never proves a different cell size and fires on sound content in a known direction — an 8x8 sheet holding 56 authored frames, an alpha-faded flipbook whose last row is fully transparent — and that direction is now stated in the issue message and in the wiki's limitation block rather than left for a reader to discover. The same reasoning shaped the detector's slice guard: drawn slices must form a contiguous prefix of at least two, and TRAILING blank slices are allowed, or the true grid of an underfilled sheet would be rejected and one of its divisors accepted in its place — a confidently wrong period, which is exactly what an error-grade signal must not produce. **(3) The two codes were exclusive.** Removed the `continue` in `AddSubUVAtlasIssues`: a renderer whose name or dimensions disagree AND whose pixels could not be read now emits both `_MISMATCH` and `_INDETERMINATE`, because dropping the second reported a heuristic finding as if the atlas had been measured. Files: `Source/PinWright/Private/Handlers/Niagara/NiagaraSubUVAtlasCheck.h/.cpp`, `.../NiagaraInspectHandler.cpp`, `Docs/wiki-src/niagara.md`. Test `PinWright.niagara.validate.SubUVAtlasMismatch` gained a **packed** fixture — a 48x48 sheet drawn as a full 6x6 of blobs with visible gutters, declared 8x8 — asserting `detectedGrid == (6,6)`, `HasDetectedGridMismatch()`, and that every weak signal is silent on it (`emptyTileCount == 0`, `PopulatedGrid == (8,8)`, divisible, no name grid). Counterfactual: remove the period detection and that fixture's `DetectedGrid` stays (0,0), `IsMismatch()` is false, and the reported defect produces no finding at all. Also added: the same packed atlas declared 6x6 raises nothing; a 6x6 sheet with its last cell row undrawn detects 6x6 (no error) while still firing the warning-grade trailing-empty signal; and `DetectAxisDivisions` is driven directly on a synthetic six-cell profile (returns 6, confidence 1.0) and a flat one (returns 0). The `#2` limitation is narrowed, not removed: a gutterless atlas whose frames touch edge to edge still has no readable period, so `consistent` continues to mean "nothing measurable disagrees".
