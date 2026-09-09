---
id: F-texture-pixel-stats-region
title: "texture.get_pixel_stats is whole-mip only — no rect/tile-grid parameter, so a SubUV atlas cannot be measured per frame and no verb returns any sub-region pixel content"
status: IN-REVIEW
severity: High
category: feature
tags: [texture, get-pixel-stats, region, rect, tile-grid, subuv, atlas, flipbook, readback, niagara, vfx]
encounters: 3
costly: 3
lastSeen: 2026-09-08
---

# Pixel stats cover the whole image or nothing

`texture.get_pixel_stats` (shipped for `F-texture-pixel-stats-readback`) returns
per-channel mean/min/max, `grayscale`, and a content hash for one mip. It takes
`assetPath` and `mip` and nothing else, so every question of the form *"what is in
this part of the image"* is unanswerable through the plugin.

That covers the entire class of tiled textures — SubUV flipbook atlases, sprite
sheets, channel-packed masks, trim sheets, texture arrays flattened into a grid —
where the whole-image mean is close to meaningless. On a 6x6 fire flipbook the
whole-mip answer is `mean 31.9 / 255, grayscale true, alpha constant 255`: three
true facts, none of which says whether the ninth frame has a bright core, or how
many frames there even are.

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-07

```
texture.get_pixel_stats {assetPath: "/Game/ExampleContent/Effects/Textures/T_Fire_subUV_01", mip: 0}
-> {"width":1024,"height":1024,"sourceFormat":"TSF_BGRA8","pixelCount":1048576,
    "mean":{"r":31.92,"g":31.92,"b":31.92,"a":255},"min":{"r":0,...,"a":255},
    "max":{"r":255,...,"a":255},"maxChannelSpread":0,"grayscale":true,"hash":"3bcbf195"}
```

Useful, and it settled two sub-questions outright (the atlas is grayscale, so a
material deriving opacity from `max(R,G,B)` is the only option; the source alpha is
constant 255, so no alpha mask exists to recover by changing the compression
setting). It cannot answer the question the session was actually about: **is this a
6x6 atlas or an 8x8 one**, and **what does the frame at normalized age 0.22 contain**.

## Workaround used, and why it should not be the answer

I extracted the texture's 256x256 editor thumbnail out of the `.uasset` bytes
(`d.find(b'\x89PNG\r\n\x1a\n')`, then a hand-written stdlib zlib + PNG unfilter),
took per-cell means over both candidate grids, and measured gutter periodicity in
the row and column luminance profiles. That worked — 5 interior gutters at ~43 px
over 256 px in both axes, so 6x6 — and it is a bad answer for three reasons: it
reads a *thumbnail*, not the texture (4x downsampled, so peaks are averaged away and
it is stale if nobody re-thumbnailed after an edit); it depends on the `.uasset`
byte layout; and it is a hundred lines of PNG decoding to ask "what is in this
rectangle".

## What is wanted

`texture.get_pixel_stats` gains either or both of:

- `region: {x, y, width, height}` in source-mip pixels — the same stats block over a
  sub-rect. Everything else follows from this one parameter.
- `tileGrid: {columns, rows}` — one stats block per cell, so an atlas is one call
  rather than 36. Pair it with the existing whole-image block so the caller can see
  both. This is what `F-niagara-validate-subuv-atlas-vs-subimagesize` would consume
  to check a declared `SubImageSize` against the texture that is actually assigned.

A grid-detection helper (report the measured cell periodicity from the row/column
profiles) would be strictly better than either, and is the thing a caller actually
wants when the declared grid is the value under suspicion. It is cheap: one pass over
a mip, two 1D profiles, gutter spacing.

## History
- `#1-filed` `OPEN` reporter — Filed while diagnosing a four-round "the explosion has no fireball" defect. Root cause turned out to be a Niagara sprite renderer declaring `SubImageSize (8,8)` over a texture that is a 6x6 / 36-frame atlas; establishing the real grid needed sub-region pixel content, which no verb returns. `texture.get_pixel_stats` was called and worked exactly as documented (whole-mip stats over `T_Fire_subUV_01`) and its answer could not discriminate the two grids. Distinct from `F-texture-pixel-stats-readback` (IN-REVIEW), which asked for pixel stats to exist at all and is satisfied by the shipped verb; this asks for them to be addressable below the whole image. Ripgrep for `region` / `rect` / `tile` alongside `get_pixel_stats` found no existing ticket.
- `#2-second-surface-atlas-band-unverifiable` `OPEN` WEAPONS-critic — Second independent encounter, same round the ticket was filed but a different asset class and a different question; `encounters` 1 → 2, status and severity unchanged. Measured in a WEAPONS critic review round 5 on `T_WPN_AOAtlas_M`, a **2048×8192** AO atlas laid out as **16 rows of 512 px**. The only statistic `texture.get_pixel_stats` returns for it is the whole-image mean, `245.74/255` = **96.4 % white**, which is a true fact about the sheet and says nothing about any row in it. The question actually being asked — *is this particular band empty* — had to be answered by reading the producer script that generated the atlas and cross-checking against unlit captures, i.e. by inference from two sources neither of which is the texture. That is the same shape as `#1`'s SubUV case (a tiled sheet whose whole-image mean is meaningless) on a non-flipbook layout: a channel-packed / row-tiled AO atlas, where the tiles are bands rather than a square grid. Note for whoever implements this: the `region {x, y, width, height}` form asked for in `#1` covers this case exactly and `tileGrid {columns, rows}` covers it too (as `{columns:1, rows:16}`), so this encounter adds no new parameter to the ask — it widens the evidence from VFX flipbooks to any tiled authoring texture, and confirms the workaround cost is the same both times: hand-decoding pixels outside the plugin to ask what is in a rectangle. Re-searched the board before appending: `get_pixel_stats` / `pixel_stats` returns this ticket and `F-texture-pixel-stats-readback` (the shipped whole-image verb) and no other owner. Verified against this checkout's generated wiki that the verb still takes only two parameters: `X:/src/unreal/EAContentExamples58/Saved/PinWright/wiki/texture.get_pixel_stats.md` lists `assetPath` (required) and `mip` (optional, default 0) and nothing else. No plugin source was opened for this entry.
- `#3-third-encounter-whole-review-decoded-offline` `OPEN` WEAPONS-critic — Third encounter (WEAPONS critic review round 6, 2026-09-08), the same verb shape as `#1` and `#2`; `encounters` 2 -> 3, `costly` 2 -> 3, severity bumped **Medium -> High** by the cost modifier (see `#4`). Re-verified before appending that the verb is unchanged: this checkout's generated wiki `X:/src/unreal/EAContentExamples58/Saved/PinWright/wiki/texture.get_pixel_stats.md` still lists `assetPath` (required) and `mip` (optional, default 0) and nothing else — no `region`, no `tileGrid`, no grid detection. Two tiled weapon atlases this round: `T_WPN_AOAtlas_M2` is **2048x8192 laid out as 16 rows of 512 px**, `T_WPN_Markings_M2` is **2048x4096 in 8 rows**. **The cost here is larger than `#2`'s and is the reportable part: every atlas number in `Docs/fps/reviews/weapons-review-06.md` was produced by an offline stdlib decode of the PNG sources under `Docs/fps/data/textures/`, not read off the shipped `.uasset`** — an entire analysis path rebuilt outside the plugin for a whole review round, not a one-off lookup. What that path produced, none of it reachable through the verb (verb answer -> offline answer): whole-atlas mean **245.7382 -> 205.1182**; texels below 0.85 **9.03 % -> 27.22 %**; below 0.60 **2.23 % -> 21.37 %**; per-tile contact readings — slide-over-frame tile 7 **238.978 -> 92.213**, frame-under-slide tile 10 **244.719 -> 105.093**, magazine flanks tiles 0/1 **252.449 / 252.445 -> 190.624 / 190.720**, magazine top tile 3 **241.411 -> 79.621**; and the deck-strip series that root-caused the round's ADS-flatness finding — slide deck **253.524 -> 252.384**, AR rail deck **189.503 -> 187.929**, pistol frame top **249.562 -> 15.000, flat**. That last row is the sharpest case yet for the ask: a **per-strip** statistic is what identified a constant-value region inside a sheet whose whole-image mean is 205, and the whole-mip block cannot express "flat at 15.0 over this strip" in any form — a reviewer reading only the verb's answer would have shipped the flatness unexplained. **Adds no new parameter to the ask:** `region {x, y, width, height}` from `#1` covers every number above, and `tileGrid {columns, rows}` covers both atlases as `{columns: 1, rows: 16}` and `{columns: 1, rows: 8}`. What it widens is the evidence, a third time and in a new direction — VFX flipbook (`#1`), AO band sheet (`#2`), and now a case where the plugin was bypassed **entirely** for a review round's texture analysis rather than supplemented. Re-searched the board before appending: `pixel_stats` board-wide returns this ticket and `F-texture-pixel-stats-readback` (the shipped whole-image verb) and no other owner. No plugin source was opened for this entry.
- `#4-bumped-by-cost` `OPEN` WEAPONS-critic — Severity **Medium -> High** under the cost modifier (README § Severity Levels, added 2026-09-07). `costly` reached **3** at `#3`, from more than one independent task/stream, and the three entries counted are: `#1` (VFX/Niagara, the four-round "the explosion has no fireball" diagnosis — a hand-written stdlib zlib + PNG unfilter over a thumbnail extracted from `.uasset` bytes, to establish whether a grid was 6x6 or 8x8); `#2` (WEAPONS critic round 5 — an AO band answered by reading the producer script and cross-checking unlit captures, i.e. inferred from two sources neither of which is the texture); `#3` (WEAPONS critic round 6 — the round's entire atlas analysis rebuilt as an offline stdlib PNG decode). Each records a workaround far past the ten-calls / ten-minutes line and none was resolved by a `Read`, a wiki lookup or a retry, so all three qualify as costly; `#2` and `#3` are separate review rounds on different assets asking different questions, and `#1` is a different subsystem entirely. **This is the cap.** The impact class is unchanged — a missing sub-region parameter is a soft blocker with an expensive workaround, not a crash and not data loss — so `High` is the ceiling and no further bump is available from cost however many more encounters land. No status change and no change to the ask: `region` / `tileGrid` as written in `#1` still covers every encounter recorded here.
- `#5-region-and-tilegrid-shipped` `IN-REVIEW` developer — `texture.get_pixel_stats` now takes `region {x, y, width, height}` (in the read mip's pixels; `x`/`y` default to 0, the rect is clamped to the mip and echoed back as `region`, and an origin *outside* the mip is refused rather than clamped to nothing) and `tileGrid {columns, rows}` (one stats block per tile in `tiles`, row-major, tile boundaries taken from the running fraction so an area that does not divide evenly still covers every pixel exactly once, capped at 4096 tiles, refused when a tile would be empty). The two compose, so a sub-rectangle can be tiled; a row-banded sheet is `{columns: 1, rows: N}`, which is what `#2` and `#3` asked for. `Source/PinWright/Private/Handlers/Asset/TexturePixelStats.h/.cpp` gained `FPixelRegion` / `FRegionStats` / `FPixelStatsOptions`, a second `BuildPixelStatsJson` overload and a C++ `MeasureTileGrid` entry point; the whole-mip accumulate loop was factored into one `AccumulateRegion` both paths share, and the source-mip lock became RAII so a rejected region cannot leave it held. `Source/PinWright/Private/Handlers/Material/TextureHandler.cpp` parses the two objects and declares them in `RPC_PARAMS` (the dispatcher's unknown-parameter gate would otherwise refuse them). The predecessor `F-texture-pixel-stats-readback` readback path was extended, not forked. **With neither argument the response is byte-identical to the shipped whole-mip one**: `region` / `tileGrid` / `tiles` are appended after `hash`, never inserted, and `hash` still covers the whole mip so its meaning never depends on the request shape; `width`/`height` stay the mip's size (a caller needs to know what its rect was clamped against) while `pixelCount` is always the pixels the stats cover. Wiki: `Docs/wiki-src/texture.md` gained a `### texture.get_pixel_stats` section. Regression test `PinWright.texture.get_pixel_stats.RegionAndTileGrid` (`Source/PinWright/Private/Tests/Assets/TestTexturePixelStatsRegion.cpp`) builds a 4x4 image of four flat 2x2 quadrants — (10,20,30), 80, 120, 200 — whose whole-mip means are 102.5 / 105 / 107.5, none of them any quadrant's. Counterfactual: revert the region/tileGrid code in `TexturePixelStats.cpp` and the top-left region's `mean.r` reads the whole-mip 102.5 instead of 10, the per-tile array is absent, and `MeasureTileGrid` does not exist. The test also pins clamping (a 100x100 rect at (3,3) collapses to one pixel), refusal of an origin off the mip, refusal of an 8-column grid over a 4 px-wide mip, and that a default call emits none of the three new fields. Not implemented and not claimed: the grid-DETECTION helper `#1` calls "strictly better" (measured cell periodicity from the row/column luminance profiles) — `tileGrid` answers "what is in cell N", and a caller must still supply the grid.
- `#6-lock-reuses-fscopedmiplock` `IN-REVIEW` developer — Verifier follow-up on `#5`. `TexturePixelStats.cpp`'s bespoke `FLockedMip` RAII was a second implementation of `TextureSourceMip::FScopedMipLock` (`Source/PinWright/Private/Handlers/Asset/TextureSourceMipLock.h`), which already exists because of two Critical lock-leak tickets and which additionally runs `FTextureCompilingManager::FinishCompilation` before locking. Replaced it: `BuildPixelStatsJson` and `MeasureTileGrid` now take a `FScopedMipLock(Texture, TEXT("source"), /*bReadOnly=*/true, EFormatPolicy::Bgra8OrG8, MipIndex)` and read through a non-owning `FMipView`, and the whole-mip hash length comes from `Lock.GetMipSizeBytes()` instead of a second `CalcMipSize` call. Net effect beyond the dedupe: `texture.get_pixel_stats` no longer returns a spurious `PIXEL_STATS_UNAVAILABLE` for a texture whose async DDC build holds a read lock on the same `FTextureSource` — the flush is now on the path. The mip-range, source-validity and format checks moved into the lock (the format allowlist was already shared, `FScopedMipLock` calls `TexturePixelStats::IsReadableSourceFormat` for `Bgra8OrG8`), so the refusal *messages* change — they now name the asset, the mip and whether retrying helps — while the refusal *conditions* do not. The no-argument JSON is unchanged: `width`/`height` still come from `Max(1, SourceSize >> Mip)`, `sourceFormat` from `Source.GetFormat()`, `hash` from a CRC32 over `CalcMipSize(Mip)` bytes, and the field insertion order is untouched. `PinWright.texture.get_pixel_stats.RegionAndTileGrid` and the predecessor's `.SourceMipStats` both assert only that a refusal is non-empty, not its text, so neither needed changing.
