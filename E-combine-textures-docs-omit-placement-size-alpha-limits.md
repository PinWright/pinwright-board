---
id: E-combine-textures-docs-omit-placement-size-alpha-limits
title: "The texture.combine_textures page documents seven parameters and none of its three behavioural limits — no placement, row-misaligned and partially unwritten output on mismatched sizes, overlay alpha discarded — so it reads as a compositor and a caller will plan a pipeline on it"
status: DONE
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
- `#2-docs` `IN-REVIEW` developer — Verified all three claims against `TextureHandler.cpp` (`combine_textures` block): no placement params; output sized from the base's source, loop bounded by `FMath::Min3` of the three pixel counts with flat `i * 4` indexing (no stride reconciliation); RGB-only blend with `OutData[Idx + 3] = BaseData[Idx + 3]`. One correction to the filing: the unblended tail is not uninitialised — `FTextureSource::InitLayered` with null data fills via `FUniqueBuffer::AllocZeroed` (`Texture.cpp` ~2348), so it is deterministically transparent black (still not the base's pixels). Also found and documented: an unrecognised `blendMode` silently falls through to `Normal`; both inputs must be `TSF_BGRA8` (`FScopedMipLock` `Bgra8Only`); blend is on stored sRGB bytes. Docs-only fix as briefed — no refusal of size mismatches added (the ticket's "better still" option remains open if wanted). Changes: `docs/wiki-src/texture.md` new `### texture.combine_textures` Notes section (full-frame blend not compositor, no placement, same dimensions required, width-mismatch diagonal smear, fewer-pixel overlay leaves zero-filled tail, overlay alpha ignored / base alpha copied / opacity the only transparency control, other limits, readback via `texture.get_pixel_stats`); `TextureHandler.cpp` `REGISTER_TEXTURE_ACTION_HANDLER("texture.combine_textures", ...)` summary + `baseTexture`/`overlayTexture`/`blendMode`/`opacity` descriptions restated. Test: `PinWright.infra.wiki_handler.MethodPage.TextureCombineTexturesLimits` (`Source/PinWright/Private/Tests/Infra/TestTextureCombineTexturesDocs.cpp`) renders the method page via `WikiHandler::RenderPage` and asserts each limit's marker text, plus the blendMode fallback from the param description.
- `#3-review-fixes` `IN-REVIEW` developer — Applied review ad8f3e5b0f2a13074. `docs/wiki-src/texture.md` (`### texture.combine_textures`): new paragraph "The output must not name an existing asset" (output is created before the input format checks with no existing-asset guard: an existing texture there is re-created in place, an input named as the output is wiped before it is read, a refused call leaves an empty output asset); "(sRGB-encoded) bytes" replaced by "stored 8-bit bytes as-is, with no colour-space conversion (the output is always flagged sRGB)"; the zero-filled tail qualified as verified on UE 5.8 only. `TextureHandler.cpp`: `name` param description warns against naming an existing asset. `Tests/Infra/TestTextureCombineTexturesDocs.cpp`: 6 new assertions on that text (one a `TestFalse` on "sRGB-encoded"), comment fixed (section sits under `## See also`, not `## Notes`). Test `PinWright.infra.wiki_handler.MethodPage.TextureCombineTexturesLimits`. The data loss itself is filed as B-combine-textures-output-overwrites-input.
- `#4-rereview-fixes` `IN-REVIEW` developer — Applied the G35b re-review. `texture.md` output paragraph now also says a loaded asset of any other class at the output name crashes the editor (engine Fatal, `UObjectGlobals.cpp:3517-3534`); `overlayTexture` param description qualifies "transparent black" with "(UE 5.8)"; `TestTextureCombineTexturesDocs.cpp` gains an assertion on the crash clause (and the empty-output assertion follows the re-wrapped line). B-combine-textures-output-overwrites-input raised to Critical.
- `#5-verified-linux` `DONE` tester — Doc commit 7b0121d2. Passed non-skipped in run3/full: `PinWright.infra.wiki_handler.MethodPage.TextureCombineTexturesLimits` (renders the method page and asserts each limit's text). Doc verified at 7230b41d in `docs/wiki-src/texture.md` `### texture.combine_textures`. Each asked bullet is present: (1) "full-frame blend, not a compositor", no placement, consumed from pixel 0; (2) use same dimensions; a width mismatch gives a diagonal smear and a fewer-pixel overlay leaves a zero-filled tail, not the base (with the filing's "uninitialised" corrected to transparent black on UE 5.8); (3) overlay alpha ignored, base alpha copied, `opacity` the only transparency control. It also documents the blendMode fallback and the existing-output hazard. Limit: the ticket's optional "better still" refusal of size mismatches was not implemented (docs-only fix); the output-overwrite data loss is tracked in B-combine-textures-output-overwrites-input.
