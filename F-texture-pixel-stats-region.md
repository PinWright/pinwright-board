---
id: F-texture-pixel-stats-region
title: "texture.get_pixel_stats is whole-mip only — no rect/tile-grid parameter, so a SubUV atlas cannot be measured per frame and no verb returns any sub-region pixel content"
status: OPEN
severity: Medium
category: feature
tags: [texture, get-pixel-stats, region, rect, tile-grid, subuv, atlas, flipbook, readback, niagara, vfx]
encounters: 2
lastSeen: 2026-09-07
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
