---
id: F-texture-cannot-author-letterforms-or-pixel-data
title: "No texture.* verb can author letterforms, glyphs, arbitrary shapes or user-supplied pixel data — create_pattern_texture offers five fixed patterns and nothing accepts raw pixels, so a marking / stencil / label texture cannot be produced through the texture namespace at all"
status: DONE
severity: Medium
category: feature
tags: [texture, create_pattern_texture, combine_textures, annotate, render-target, glyph, letterform, text, stencil, decal, marking, pixel-data, missing-capability]
encounters: 1
costly: 1
lastSeen: 2026-09-05T19:54:25Z
---

# The texture namespace can filter an image but cannot draw one

## The gap

`texture.*` (25 verbs) is a **transform** pipeline: every verb either creates one of a small set of
parametric fills or edits an image that already exists. Nothing in it accepts a shape, a path, a
glyph, or a buffer of pixels from the caller. So a marking, stencil, label, serial number, warning
triangle or any other authored graphic cannot be produced inside the editor.

Specifically:

- `texture.create_pattern_texture` supports exactly five `patternType` values — `Checker`, `Grid`,
  `Brick`, `Stripes`, `Dots` (`Saved/PinWright/wiki/texture.create_pattern_texture.md`). There is
  no `custom`, `data`, `pixels`, `mask`, `shape` or `polygon` type, and no parameter anywhere on
  it that takes caller-supplied content.
- The other creators are the same shape: `create_gradient_texture`, `create_noise_texture`,
  `create_normal_from_height` — parametric or derived, never authored.
- There is **no** `set_pixel`, no `write_texture`, no raw-buffer verb, and no draw-to-render-target
  verb anywhere in the wiki. `texture.create_render_target` and `render.create_render_target`
  create a target that nothing can subsequently draw into.
- The remaining 20 verbs are filters and settings (`invert`, `desaturate`, `blur`, `sharpen`,
  `adjust_levels`, `adjust_curves`, `channel_pack`, `channel_extract`, `resize_texture`,
  `combine_textures`, plus the `set_*` sampling settings and the two readbacks).

## The only text rasteriser in the plugin is not usable as a source

`image.annotate` is the sole glyph renderer anywhere in PinWright, and it is built for measurement
overlays, not for authoring:

- The font is a **3x5 pixel cell** —
  `Plugins/PinWright/Source/PinWright/Private/Handlers/Render/BitmapPaint.h:118-119`
  (`GlyphCellWidth = 3`, `GlyphCellHeight = 5`).
- The character set is `0-9 A-Z . - : /`, case-folded — no lowercase exists.
- **Every label is drawn on a mandatory opaque dark backing plate.**
  `BitmapPaint.h:138-146`: `FLabelStyle` declares `FPaint Plate = FPaint(FColor(16, 16, 16, 255))`
  with `bool bDrawPlate = true`, and the RPC exposes no parameter to disable it — `labels[]` takes
  `{x, y, z?, text, color?, opacity?, scale?}` and nothing else
  (`Saved/PinWright/wiki/image.annotate.md`).

A 3x5 uppercase glyph on a forced opaque plate is correct for an annotation and unusable as a
stencil source: the plate is the mask's own background punched out as a solid rectangle.

`image.annotate` also operates on a **file**, positioned in **world** coordinates via a
georeference, and writes a PNG — not a `UTexture2D`. It is not on the texture path at all.

## The only routes to a letterform both leave the editor

1. Hand-author a PNG outside the editor, then `asset.import`.
2. Build a UMG widget, `widget.screenshot_designer` to a PNG, then `asset.import`.

Both do a texture job outside the texture namespace, and (2) additionally runs through the widget
designer, which `Docs/fps/PLAN.md` rule 10 forbids screenshotting for unrelated crash reasons.

## Workaround actually used

Route 1, written from scratch because nothing existed to reuse: a stroke-font rasteriser plus an
8-bit-grayscale PNG encoder in stdlib Python (`zlib` + `struct`), at
`X:/src/unreal/EAContentExamples58/Docs/fps/scripts/wpn_author_markings.py` — 288 lines, including
a hand-plotted 41-glyph vector font — purely because the plugin cannot draw a letter. That producer
is the evidence: it exists only to fill this gap, and every project that wants a labelled surface
will write its own copy of it.

## What is asked for

Either or both:

- **`texture.create_text_texture`** — `text`, `font`, `size`, `align`, `color`,
  `backgroundColor`, `width`, `height`. Real letterforms at an authorable size, with a
  **transparent** background available rather than a forced plate.
- **`texture.draw`** — a list of primitives (`line`, `rect`, `circle`, `polyline`, `text`) each
  with an (x, y) placement, composited alpha-aware into a target texture. This is the general
  answer and subsumes the first.

## `combine_textures` is not a substitute for compositing

Worth stating explicitly, because it is the verb a reader would reach for and it will not do the
job (`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp:2263-2385`):

- **No placement.** The parameter list is `baseTexture`, `overlayTexture`, `blendMode`, `opacity`,
  `name`, `path`, `save` — no offset, origin, rect or anchor of any kind.
- **Mismatched sizes misalign, they do not letterbox.** The blend walks a single flat linear pixel
  index bounded by `FMath::Min3` of the base, overlay and output pixel counts (`:2343-2347`) and
  indexes as `i * 4`. Rows are never reconciled, so an overlay narrower than the base advances one
  scanline out of phase per row and lands as a diagonal smear; and because the loop stops at the
  minimum, the remainder of the output is never written by the blend at all — it keeps whatever
  `FTextureSource::Init` left there, not the base's pixels.
- **Overlay alpha is ignored and output alpha is forced to the base's.** `:2380` is
  `OutData[Idx + 3] = BaseData[Idx + 3];` and the blend loop runs `c` over 0..2 only, so an alpha
  stencil composites as if fully opaque.

So a small stencil cannot be stamped into a larger sheet. Filed as a documentation gap in its own
right: `E-combine-textures-docs-omit-placement-size-alpha-limits`.

severity rationale: impact=an entire authoring category (any marking, label, stencil or decal
source) has no in-editor route and forces a hand-written external producer x reach=any project
needing text or an arbitrary shape on a surface -> Medium

## History
- `#1-filed` `OPEN` reporter — Filed from the FPS WEAPONS stream while authoring weapon markings. Exhaustive read of the 25 `texture.*` wiki pages: `create_pattern_texture` offers five fixed `patternType` values (`Checker`, `Grid`, `Brick`, `Stripes`, `Dots`) with no `custom`/`data`/`pixels`/`mask`/`shape`/`polygon` type; the other creators are parametric or derived; there is no `set_pixel`, no `write_texture`, no raw-buffer verb and no draw-to-render-target verb anywhere in the wiki (`texture.create_render_target` and `render.create_render_target` make a target nothing can draw into); the rest are filters and sampling settings. The plugin's only glyph rasteriser is `image.annotate`, whose font is a 3x5 pixel cell (`BitmapPaint.h:118-119`, `GlyphCellWidth = 3` / `GlyphCellHeight = 5`), whose character set is `0-9 A-Z . - : /` case-folded with no lowercase, and whose every label sits on a mandatory opaque dark plate (`BitmapPaint.h:138-146`: `Plate = FColor(16,16,16,255)`, `bDrawPlate = true`) that no RPC parameter disables — `labels[]` takes only `{x, y, z?, text, color?, opacity?, scale?}`. It also works on a file in world coordinates and emits a PNG, not a `UTexture2D`. The only routes to real letterforms leave the editor: hand-author a PNG + `asset.import`, or UMG + `widget.screenshot_designer` + `asset.import`. Workaround actually used, and the evidence the gap is real: a stroke-font rasteriser and an 8-bit-grayscale PNG encoder written from scratch in stdlib Python (`zlib` + `struct`) at `Docs/fps/scripts/wpn_author_markings.py`, 288 lines including a hand-plotted 41-glyph vector font, purely because the plugin cannot draw a letter. Ask: `texture.create_text_texture` (`text`, `font`, `size`, `align`, `color`, `backgroundColor`, `width`, `height`) with a transparent background available, and/or the general `texture.draw` taking placed `line`/`rect`/`circle`/`polyline`/`text` primitives composited alpha-aware. `texture.combine_textures` is explicitly not a substitute — no placement parameter, a flat linear index bounded by `FMath::Min3` (`TextureHandler.cpp:2343-2347`) that misaligns rows on mismatched sizes and leaves the output remainder unwritten, and output alpha forced to the base's at `:2380` with the blend running channels 0..2 only.
- `#2-text-and-pixel-creators` `IN-REVIEW` developer — Added two authoring verbs in a new `Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureAuthorHandler.cpp`. **`texture.create_text_texture`**: `text` (`'\n'` = new line), `font` (any runtime `UFont` composite or `UFontFace`, default `/Engine/EngineFonts/Roboto.Roboto`), `typeface` (composite entry, e.g. Regular/Bold/Italic/Bold Italic/Light; unknown refused with the list; offline bitmap fonts refused), `size` (pixel em 1..1024), `align` (Left/Center/Right, unknown refused), `color`, `backgroundColor` (default fully transparent — text colour on every pixel, coverage as alpha, no plate), optional `width`/`height` (default: fit to the line block). Glyphs are rasterised on the CPU by FreeType from the font's own face bytes (kerning applied; no HarfBuzz shaping — complex scripts are out of scope). Response measures rather than echoes: `inkPixels`, `inkBounds`, `clipped`/`clippedPixels` (coverage that fell off the canvas), `missingCharacters` (U+XXXX the font lacks). **`texture.create_from_pixels`**: `width`, `height` (1..4096), `data` base64 raw bytes row-major from the top-left, `format` `RGBA8` (default) or `Gray8`; byte count must equal width*height*bpp exactly or `INVALID_ARGUMENT` names both counts; bounded in practice by the 1 MiB request body. Both refuse an existing asset with `ASSET_ALREADY_EXISTS` (never NewObject over a live object), reuse the shared `CreateEmptyTexture` / `McpSaveTextureToDisk` (made non-static in `TextureHandler.cpp`, no other change there) and return `pixelStats` read back from the written source mip. `texture.draw` (placed primitives into an existing texture) was NOT built: create_from_pixels covers arbitrary shapes by caller rasterisation; stamping into an existing sheet still needs `combine_textures` placement (`E-combine-textures-docs-omit-placement-size-alpha-limits`). `PinWright.Build.cs` gains `AddEngineThirdPartyPrivateStaticDependencies(Target, "FreeType2")` (same dependency SlateCore and the engine's Text3D plugin take). Docs: `docs/wiki-src/texture.md` prelude + `### texture.create_from_pixels` / `### texture.create_text_texture`. Tests (`Tests/Material/TestTextureAuthorVerbs.cpp`, filter `PinWright.texture.create_from_pixels+PinWright.texture.create_text_texture`): `PinWright.texture.create_from_pixels.WritesCallerBytesExactly` (every RGBA8/Gray8 byte read back from the source mip at its own pixel), `PinWright.texture.create_from_pixels.RefusesMismatchedOrUnknownInput` (short data, unknown format, oversize, existing name — nothing created or replaced), `PinWright.texture.create_text_texture.DrawsLetterformOnTransparentBackground` ("L" at 96px: ink > 40px tall, solid stem and foot, empty upper-right quadrant, red on every inked pixel, >50% of canvas alpha 0, response ink measurements equal readback), `PinWright.texture.create_text_texture.OptionsReachTheRaster` (Bold inks >10% more than Regular, Left/Right land in opposite quarters of a 512px canvas, opaque background leaves no transparent pixel, 20px canvas reports clipped, unknown typeface refused).
- `#3-review-fixes` `IN-REVIEW` developer — Applied the independent review (2 SHOULD-FIX + TextureHandler.cpp hunk). (1) `create_text_texture` refuses `text` over 4096 characters with `INVALID_ARGUMENT` right after `RequireString` (a 1 MiB text was ~2M `FT_Load_Glyph` calls on the game thread), and the destination (name/path/existing asset) is now validated before the font load and raster instead of after. (2) Auto-size fitted the advance, not the ink: an italic overhang (Roboto Italic `f` +12px at 128px) or a leading `j` (lsb -4px) was clipped. The layout pass now tracks each line's ink extent from the hinted glyph metrics (pen floored, bitmap box floored/ceiled - the raster's own rounding); a line's box is its advance plus any ink outside it, the canvas fits the widest box and alignment places the whole box (origin shifted by the negative minimum). (3) `CreateEmptyTexture` / `McpSaveTextureToDisk` are no longer hand-forward-declared module-global symbols: they live in `namespace PinWrightTextureAssets`, declared once in the new `Handlers/Material/TextureAssetHelpers.h` that both `TextureHandler.cpp` (qualified definitions + two file-scope using-declarations so its 35 call sites stay unchanged) and `TextureAuthorHandler.cpp` (qualified calls) include. NITs: `.ttc` composite entries pass `GetSubFaceIndex()` to `FT_New_Memory_Face`; wiki says kerning is legacy `kern`-table only. Skipped: half-created asset on mip-lock failure, `Target.bCompileFreeType` gate (the handler includes FreeType headers unconditionally, so a gate needs a matching `#if` - not worth it while every supported editor target compiles FreeType). New test `PinWright.texture.create_text_texture.AutoSizeFitsInkAndBoundsInput` (auto-sized Italic `ff`, Regular `j`, Bold Italic `jf` at 128px report `clipped:false` - fails on revert of the ink fit; 4097-char text refused with no asset; second `create_text_texture` under an existing name refused `ASSET_ALREADY_EXISTS`; `AB\n<U+E000>` reports 2 lines, twice one line's height, `missingCharacters:["U+E000"]`); `RefusesMismatchedOrUnknownInput` gains a non-base64 `DECODE_FAILED` case and a `TestNotNull` on the kept texture. Filter: `PinWright.texture.create_from_pixels+PinWright.texture.create_text_texture`. fastcheck OK on all three .cpp plus a two-file unity probe; check_test_ids / check_test_skips clean. Not built or run yet.
- `#4-re-review-fixes` `IN-REVIEW` developer — The 4097-character refusal case in `PinWright.texture.create_text_texture.AutoSizeFitsInkAndBoundsInput` passed even with the cap reverted, because the auto-size range check refused first. It now pins a 64x64 canvas and asserts INVALID_ARGUMENT, a message naming "4096", and that no asset is created. The test comment no longer claims an ordering the test cannot observe. fastcheck OK; check_test_ids and check_test_skips clean.
- `#5-verified-linux` `DONE` tester — Fix commit ab30256d. Passed non-skipped in run3/full: `PinWright.texture.create_text_texture.{DrawsLetterformOnTransparentBackground, OptionsReachTheRaster, AutoSizeFitsInkAndBoundsInput}` and `PinWright.texture.create_from_pixels.{WritesCallerBytesExactly, RefusesMismatchedOrUnknownInput}`. Acceptance ("either or both"): `texture.create_text_texture` takes all of the asked parameters (`text`, `font`, `size`, `align`, `color`, `backgroundColor`, `width`, `height`) and renders real FreeType letterforms with a transparent background and no plate. The tests check the ink shape, the typeface and alignment effects, an opaque background, and clipping and missing-glyph reporting. `texture.create_from_pixels` also covers arbitrary shapes via caller-supplied RGBA8/Gray8 bytes, read back byte-exact. Limits: `texture.draw` (placed primitives into an existing texture) was not built, which the ask's "either" allows; there is no HarfBuzz shaping (complex scripts) and only legacy `kern`-table kerning; stamping into a larger sheet still depends on combine_textures, which has no placement.
