---
id: B-pwmodel-per-part-uv-layout-overlaps-across-parts
title: "`uv mode=layout` / `mode=patch_builder` pack to the unit square and `uv` is a PART op, so every part of a model lands on top of every other — and Examples/pwmodel/chess_rook.pwmodel ships three parts stacked on its own LIGHTMAP channel"
status: OPEN
severity: High
category: bug
tags: [pwmodel, uv, layout, patch_builder, xatlas, lightmap, examples, chess_rook, cross-part-overlap, no-diagnostic]
encounters: 1
lastSeen: 2026-09-03T04:45:00+03:00
---

# Per-part packing IS cross-part overlap, and the shipped example demonstrates it on a lightmap

`uv` is a part-scoped op. Every mode that packs — `xatlas`, `patch_builder` with `auto_pack`,
and `layout` with `layout_type` `repack` / `stack` / `normalize` — packs into the **unit
square**. Parts then merge into one mesh. So a model that runs the documented
`patch_builder` -> `layout` pair in each part ends with every part occupying 0-1 and every part
overlapping every other, and nothing reports it: there is no cross-part UV check the way there
is a cross-part geometry check (`PWMODEL_UNUNIONED_OVERLAP_PARTS`).

## The shipped example has it, on the one channel where it is unambiguously a defect

`Plugins/PinWright/Examples/pwmodel/chess_rook.pwmodel` — three parts, each ending with:

```
part body   { ... uv channel=1 mode=patch_builder
                  uv channel=1 mode=layout texture_resolution=256 }      # :63-64
part crown  { ... uv channel=1 mode=patch_builder
                  uv channel=1 mode=layout texture_resolution=256 }      # :133-134
part felt   { ... uv channel=1 mode=patch_builder
                  uv channel=1 mode=layout texture_resolution=256 }      # :150-151

lightmap channel=1 resolution=64                                          # :159
```

Three independent packs into 0-1, declared as the asset's lightmap channel. A lightmap UV set
whose islands overlap bakes one part's irradiance onto another — it is the one channel where
non-overlap is not a preference but the contract. The file's own comment at :154-155 explains
the `resolution=64` choice and says nothing about the overlap.

## The mechanism, measured

Measured on `Content/FPS/Weapons/Meshes/SM_WPN_AR` (12 parts). With
`patch_builder` + `layout repack` per part and nothing else, every part packs to 0-1. Adding a
third op per part — `uv channel=0 mode=layout layout_type=transform layout_scale=S
translation=(tx, ty)` — with a hand-assigned disjoint tile per part is what separates them.
Read back off the compiled asset via
`GeometryScript_MeshQueries.get_triangle_u_vs`, all 19,086 triangles:

```
tile handguard   tris=  284      tile barrel      tris= 1566
tile upper       tris=  424      tile grip        tris=  196
tile lower       tris=  324      tile stock       tris=  382
tile rail        tris=13106      tile buffer_tube tris=  610
tile magazine    tris=  314      tile optic_body  tris= 1450
tile trigger     tris=   46      tile optic_lens  tris=  384
unassigned=0  spanning_multiple_tiles=0  uv_outside_unit_square=0
```

Every per-tile count equals that part's `meshTriangleCount` from the compile response, so the
separation is exact — and it exists **only** because of the third op. Remove the
`layout_type=transform` lines and all twelve counts collapse onto one another.

## Why the workaround is bad enough to be worth a ticket

`layout_scale` is a single uniform number, so each part's tile must be a **square**. Packing 12
parts of wildly different surface area (9 uu2 to 950 uu2) into squares by hand means either
uniform tiles and 25x texel-density spread, or a per-part solve that has to be re-run by hand
every time the geometry changes. The atlas is arithmetic the author maintains, in comments,
with nothing checking it — exactly the class of hand-maintained invariant this format exists to
remove.

## What it should do

Any one of these closes it; the first is the smallest:

1. **A cross-part UV overlap diagnostic**, sibling to `PWMODEL_UNUNIONED_OVERLAP_PARTS`. The
   merge stage already has every part's triangles; comparing per-part UV bounding boxes on each
   populated channel is cheap, and it would have caught `chess_rook` before it shipped. On a
   channel named by `lightmap` it should be an **error**, not a warning.
2. **A model-level packing statement** — `uv_layout channel=N texture_resolution=M` outside the
   parts, run after merge, packing every part's islands into one atlas with real
   area-proportional density. That is what `layout repack` already does; it is only its scope
   that is wrong.
3. At minimum, say it in `model.authoring`: the Materials/UV guidance never mentions that a
   packing mode in a part is a per-part pack, and the one worked example that uses the pair gets
   it wrong.

Fixing `chess_rook.pwmodel` needs (2) or a hand atlas; until then its `lightmap channel=1` line
is claiming something the UVs do not support.

severity rationale: impact=every multi-part model that packs UVs ships an unusable channel, with
no diagnostic, and a shipped example demonstrates it on a lightmap x reach=all of `model.*`
-> High

## History
- `#1-initial-repro` `OPEN` reporter — Found while replacing per-part `uv mode=box` on
  `SM_WPN_AR` and `SM_WPN_Pistol` (a review called UV0 "per-part box projection with cross-part
  overlap"). `patch_builder` + `layout repack` reproduces the same overlap in a different shape,
  because both pack to the unit square inside a part. Worked around with a hand-assigned
  `layout_type=transform` tile per part (4x4 grid on the AR, 2x2 on the pistol), verified by
  reading all 19,086 + 4,814 compiled triangles back and classifying each by tile: 0 unassigned,
  0 spanning two tiles, 0 outside 0-1, and per-tile counts equal to per-part counts. While
  reading the format's own example for the `patch_builder` -> `layout` pattern, found
  `Examples/pwmodel/chess_rook.pwmodel` doing it per part on the channel its `lightmap` statement
  names. chess_rook was read from source, not compiled and measured here; the mechanism was
  measured on SM_WPN_AR.
