---
id: E-pwmodel-corpus-lightmap-gap-undeclared
title: "Nine of the thirteen .pwmodel examples ship with no lightmap UV channel and LightMapResolution 4, and the page that catalogues what the corpus does not exercise has no lightmap row"
status: OPEN
severity: Low
category: ergonomic
tags: [pwmodel, examples, lightmap, corpus-coverage, docs, op-coverage]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# The corpus does not exercise baked lighting, and does not say so

## The reported premise does not hold

The claim was that nine examples carry a `uv channel=1` unwrap that is thrown away for want of a
`lightmap` statement. Across all thirteen `Examples/pwmodel/*.pwmodel`, `uv channel=1` and the
`lightmap` statement are **perfectly correlated**. Nothing is discarded in any shipped file.

| file | `lightmap` | `uv channel=1` |
|---|---|---|
| `amphora` | `channel=1 resolution=64` (`:215`) | 2 (`:79`, `:209`) |
| `chess_rook` | `channel=1 resolution=64` (`:148`) | 6 (`:63,:64,:122,:123,:139,:140`) |
| `oil_lamp` | `channel=1 resolution=64` (`:321`) | 3 (`:254,:284,:313`) |
| `watchtower` | `channel=1 resolution=128` (`:310`) | 5 (`:138,:195,:229,:262`, +1) |
| `crystal_cluster` | — | 0 (channel 0 only: `:156,:192,:255`) |
| `driftwood` | — | 0 (`:363,:417,:457,:487,:514`) |
| `gothic_window` | — | 0 (`:213,:290,:302,:336`) |
| `mobius_band` | — | 0 (`:323`) |
| `origami_crane` | — | 0 (`:226,:291`) |
| `pipe_junction` | — | 0 (`:191,:226`) |
| `ships_wheel` | — | 0 (`:171,:189,:225`) |
| `spiral_stair` | — | 0 (`:160,:214,:239,:274,:308`) |
| `spur_gear` | — | 0 (`:234,:253`) |

The 4/9 partition in the report is right; the wasted-unwrap premise is not. The four that pay for an
unwrap are exactly the four that set a resolution to use it.

## The 4x4 default is real but correctly gated

`PwModelParser.cpp:344` — "Omitted, the asset keeps the UStaticMesh default of 4, so authored
lightmap UVs bake at 4x4." `PwModelCompiler.cpp:2526-2536` reads the lightmap spec only inside
`if (Document.Lightmap.IsSet())`, so with no statement `GeometryAssetCreate.cpp:296-305` writes
neither `LightMapResolution` nor `LightMapChannel` (both carry independent `INDEX_NONE` sentinels)
and the asset keeps `UStaticMesh`'s constructor default of 4. The compiler's own 4x4 warning at
`GeometryAssetCreate.cpp:368-378` is deliberately scoped `else if (Spec.LightMapChannel !=
INDEX_NONE)` so "a caller who never mentioned a lightmap is not nagged about one" (`:371-372`) —
correct behaviour, correctly not firing for the nine.

## What actually survives

With `Options.bGenerateLightmapUVs = false` (`GeometryAssetCreate.cpp:247`), those nine ship with
**no lightmap UV channel at all**, and `EnforceLightmapRestrictions` clamps
`LightMapCoordinateIndex` into `[0, NumUVs-1]` at build (`GeometryAssetCreate.cpp:340-342`), landing
on channel 0 — the textured, overlapping channel. They therefore cannot be lit with baked static
lighting without the consumer adding a `lightmap` statement.

That is a corpus-coverage gap, not a compiler defect: the compiler behaves as designed and as
documented (`Docs/wiki-src/model.authoring.md:69`). The gap is that
`Docs/wiki-src/model.examples.op-coverage.md` — the page whose job is to catalogue what the corpus
does **not** exercise — has no lightmap row (grep for `lightmap` there returns nothing).

**Fix:** add a row to `model.examples.op-coverage.md` stating that nine of thirteen examples author
no lightmap and bake at the `UStaticMesh` default of 4, that baked static lighting is therefore
unexercised outside the four that set one, and that the pattern to copy is `amphora:215`.

## History
- `#1-premise-inverted-coverage-gap-survives` `OPEN` reporter — The reported premise is inverted: across all thirteen `Examples/pwmodel/*.pwmodel`, `uv channel=1` and the `lightmap` statement are perfectly correlated, so no shipped example throws away an unwrap. The four with a `lightmap` statement — `amphora:215`, `chess_rook:148`, `oil_lamp:321`, `watchtower:310` — are exactly the four with channel-1 unwrap ops; the other nine (`crystal_cluster`, `driftwood`, `gothic_window`, `mobius_band`, `origami_crane`, `pipe_junction`, `ships_wheel`, `spiral_stair`, `spur_gear`) author `uv channel=0` only. The 4/9 partition in the report is correct. The default-resolution half is correct and correctly gated: `PwModelCompiler.cpp:2526-2536` reads the lightmap spec only inside `if (Document.Lightmap.IsSet())`, so with no statement `GeometryAssetCreate.cpp:296-305` writes neither field (independent `INDEX_NONE` sentinels) and the asset keeps `UStaticMesh`'s default of 4 (`PwModelParser.cpp:344`), while the compiler's 4x4 warning at `GeometryAssetCreate.cpp:368-378` is deliberately scoped `else if (Spec.LightMapChannel != INDEX_NONE)` so a document that never mentioned a lightmap is not nagged (`:371-372`). What survives is a corpus-coverage gap: with `Options.bGenerateLightmapUVs = false` (`GeometryAssetCreate.cpp:247`) the nine ship with no lightmap UV channel at all and `EnforceLightmapRestrictions` clamps `LightMapCoordinateIndex` to channel 0 (`:340-342`), so baked static lighting is unexercised by the corpus — and `Docs/wiki-src/model.examples.op-coverage.md`, the page that catalogues what the corpus does not exercise, has no lightmap row. Fix: add that row, naming `amphora:215` as the pattern to copy.
