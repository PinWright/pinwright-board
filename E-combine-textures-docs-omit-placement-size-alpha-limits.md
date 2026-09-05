---
id: E-combine-textures-docs-omit-placement-size-alpha-limits
title: "The texture.combine_textures page documents seven parameters and none of its three behavioural limits — no placement, row-misaligned and partially unwritten output on mismatched sizes, overlay alpha discarded — so it reads as a compositor and a caller will plan a pipeline on it"
status: OPEN
severity: Medium
category: ergonomic
tags: [texture, combine_textures, wiki, docs, compositing, placement, alpha, mismatched-size, silent-wrong-result]
encounters: 1
lastSeen: 2026-09-05T19:54:25Z
---

# The page describes the arguments and not the operation

`Saved/PinWright/wiki/texture.combine_textures.md` is, in full, a one-line summary ("Combine
textures") and seven parameter lines: `baseTexture`, `overlayTexture`, `blendMode`, `opacity`,
`name`, `path`, `save`. No Notes section. Nothing on the page distinguishes it from a general
compositor, and its name and blend-mode vocabulary (`Normal`, `Multiply`, `Screen`, `Overlay`,
`Add`) actively suggest one.

Three properties of the implementation
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp:2263-2385`) are
undocumented, and each of them turns a reasonable plan into a wrong image with a success response.

**1. There is no placement.** The parameter list is the whole interface: no offset, origin, rect,
anchor or scale. The overlay is applied at pixel index 0 and runs to the end. A caller cannot stamp
a stencil, a logo or a label anywhere but the top-left corner at full size.

**2. Mismatched sizes misalign the rows and leave the rest of the output unwritten.** The output is
created at the **base's** source dimensions (`:2290-2291`), and the blend then walks one flat
linear pixel index bounded by `FMath::Min3` of the base, overlay and output pixel counts
(`:2343-2347`), indexing as `i * 4`. Nothing reconciles the two row strides, so an overlay narrower
than the base advances one scanline out of phase per output row — the result is a diagonal smear,
not a corner placement. And because the loop stops at the minimum pixel count, the tail of the
output is **never written by the blend at all**: it holds whatever `FTextureSource::Init` left
there (`CreateEmptyTexture`, `:83`), not the base's pixels. So combining a small overlay into a
large base does not "leave the rest of the base intact" — it discards it.

The `Min3` clamp itself is correct and deliberate; the comment at `:2341-2342` records that it was
added because a smaller overlay used to read past its own end. The gap is that the caller is never
told what the clamp does to their image.

**3. Overlay alpha is discarded, and the output takes the base's alpha.** The blend loop runs `c`
over channels 0..2 only, and `:2380` is `OutData[Idx + 3] = BaseData[Idx + 3];`. A transparent
overlay therefore composites as if fully opaque — `opacity` is a uniform scalar over the whole
image and is the only transparency the verb honours.

## Why this is worth a page edit rather than a shrug

All three failures are silent: the call returns success and produces a plausible-looking asset. The
caller who trusts the page plans "author a small marking, stamp it onto the sheet", gets a diagonal
smear over a partly-uninitialised sheet, and has no readback that would flag it — the metadata
verbs confirm only the envelope (cf. `F-texture-pixel-stats-readback`, which added
`texture.get_pixel_stats` for exactly this class of unverifiability; it reports aggregates, not
placement).

## Asked for

A Notes section on `Docs/wiki-src/` for `texture.combine_textures` stating, plainly:

- the verb is a **full-frame blend**, not a compositor: there is no placement, and both images are
  consumed from pixel 0;
- both textures should be the **same dimensions**; on a mismatch the blend is row-misaligned and
  the output's tail is left uninitialised rather than falling back to the base;
- the **overlay's alpha is ignored** and the output's alpha is copied from the base; `opacity` is
  the only transparency control.

Better still, refuse a size mismatch with an explicit error rather than documenting the smear — the
handler already knows both sizes at `:2290` and `GetNumPixels()` at `:2344`, so the check is free,
and no correct caller depends on the current behaviour.

## Not the other combine_textures tickets

- `B-combine-textures-leaks-bulkdata-lock-then-crashes` (IN-REVIEW, Critical) — the bulk-data lock
  leak in this same block. Fixed by `FScopedMipLock`; unrelated to what the page says.
- `E-texture-action-handler-param-docs` (IN-REVIEW) — the macro-registered `texture.*` family
  rendering "Parameters: none". That is about the parameters **existing** on the page; this is
  about the seven that are there being insufficient to use the verb correctly.
- `F-texture-cannot-author-letterforms-or-pixel-data` — the authoring gap that led here.

severity rationale: impact=a documented verb silently produces a wrong image on a plan the page
invites x reach=any caller compositing two textures of different sizes or with alpha -> Medium

## History
- `#1-filed` `OPEN` reporter — Filed from the FPS WEAPONS stream after planning a stencil-stamping pipeline on `texture.combine_textures` and reading the source when it did not fit. The wiki page is a one-line summary plus seven parameter lines with no Notes section, and omits all three of: (a) there is no placement parameter, so the overlay is applied from pixel 0 at full size; (b) on mismatched sizes the blend walks a flat linear index bounded by `FMath::Min3` (`TextureHandler.cpp:2343-2347`) indexed `i * 4` with no row-stride reconciliation, so a narrower overlay advances one scanline out of phase per row (diagonal smear) and the output tail is never written by the blend — it keeps `FTextureSource::Init`'s contents from `CreateEmptyTexture` (`:83`), not the base's pixels, even though the output is sized from the base (`:2290-2291`); (c) the loop runs channels 0..2 and `:2380` copies the base's alpha into the output, so overlay alpha is discarded and `opacity` is the only transparency control. All three fail silently with a success response and a plausible asset. Asked: a Notes section stating it is a full-frame blend rather than a compositor, that both inputs should share dimensions, and that overlay alpha is ignored — or better, refuse a size mismatch outright, since both sizes are already known at `:2290`/`:2344` and no correct caller depends on the smear. Distinct from `B-combine-textures-leaks-bulkdata-lock-then-crashes` (the lock leak in the same block) and `E-texture-action-handler-param-docs` (parameters missing from the page entirely, rather than present but insufficient).
