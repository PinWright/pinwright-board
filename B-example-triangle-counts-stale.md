---
id: B-example-triangle-counts-stale
title: "The mesh/asset triangle-count pairs for gothic_window, watchtower and spur_gear are hand-duplicated in five places and the mesh half of every one is stale — off by up to 10.8x"
status: OPEN
severity: High
category: bug
tags: [pwmodel, docs, stale-numbers, triangle-count, duplication, comments, pwmodel-format, examples]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# One number, copied five times by hand, wrong in all five

| # | Location | Triangle pair | Vertex pair |
|---|---|---|---|
| 1 | `Docs/pwmodel-format.md:1323` (+ `:1324`) — the normative table | gothic_window 8174 → 7956, watchtower 10840 → 10718, spur_gear 167114 → 165558 | origami_crane 50 → 162, crystal_cluster 1342 → 2180, spiral_stair 2356 → 4556 |
| 2 | `Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryAssetCreate.cpp:415-416` (+ `:418-419`) | identical | identical |
| 3 | `Source/PinWrightGeometry/Private/Handlers/Model/ModelCompileHandler.cpp:438-440` | gothic_window only | origami_crane only |
| 4 | `Source/PinWrightGeometry/Private/Model/PwModelCompiler.h:84-88` | identical | identical |
| 5 | `Source/PinWrightGeometry/Private/Tests/Model/TestPwModelCompiler.cpp:787-790` | identical | origami_crane only |

**Correction to the report: it is five locations, not six.** An exhaustive grep of the six numerals
across the tree (excluding `Binaries/`, `Intermediate/`, `dist/`, `scratchpad/`) returns exactly these
files. The likely origin of the miscount is that `pwmodel-format.md` contributes two table rows
(`:1323` triangles, `:1324` vertices). Two near-misses that are not pairs: `Docs/wiki-src/model.md:76`
states the four-count doctrine with no numbers, and `Docs/wiki-src/model.authoring.md:218` cites a
different 54 → 144 vertex jump.

## The mesh half is provably stale — no compile needed

All five copies are byte-identical, so inter-copy agreement proves nothing. The disagreement is with
freshly measured figures elsewhere in the same tree, all landed in this wave.
`Docs/wiki-src/model.examples.md:13` declares its Tris column to be `meshTriangleCount` read from
`model.validate`, **measured 2026-08-20**:

| example | comments claim | measured | factor |
|---|---|---|---|
| gothic_window | 8,174 | **4,362** (`model.examples.md:21`) | 1.9x |
| watchtower | 10,840 | **4,572** (`:29`) | 2.4x |
| spur_gear | 167,114 | **15,528** (`:28`) | 10.8x |

Three further post-wave sources agree with the low figures independently:
`Examples/pwmodel/gothic_window.pwmodel:350` ("asks for complex and gets the real 4,362 triangles"),
`Examples/pwmodel/spur_gear.pwmodel:185` ("array_radial count=2 axis=x   15,528 triangles, 0
degenerate"), and `Docs/wiki-src/model.examples.op-coverage.md:88` ("`watchtower` now compiles 4,572
triangles with 0 degenerates").

## What still needs a compile

The **asset** side (7956 / 10718 / 165558) and all three vertex pairs cannot be re-derived from
`model.validate`, which omits `assetTriangleCount` / `assetVertexCount` by design
(`Docs/pwmodel-format.md:1330-1332`, `Docs/wiki-src/model.md:76`), and `model.examples.md` publishes
no vertex column. Correcting those needs one `model.compile` per example on post-wave binaries.

## How it drifted

The numbers were duplicated by hand, and the examples were then reworked by `2988dead`, `0f2f3570`
and `37fd826c` without any of the five copies being revisited. Nothing reconciles them — see
`F-citation-checker-repo-wide` for the general gap.

## Same-family staleness alongside

`Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:2389` and
`Source/PinWrightGeometry/Private/Tests/Model/TestPwModelCompiler.cpp:2337` both state "degenerates —
6 in gothic_window, 70 in watchtower". `Docs/wiki-src/model.examples.md` says only `oil_lamp` carries
any degenerates (2), and `model.examples.op-coverage.md:88` puts watchtower at 0. Two more stale
sites, same root cause, also checkable without a compile.

**Fix:** re-measure all four counts for the three examples with one `model.compile` each on post-wave
binaries, then collapse the duplication — keep the normative table in `Docs/pwmodel-format.md` and
have the four C++ comment blocks reference it by path instead of restating numbers. A comment that
cites a table cannot drift from it silently; five hand-copies always can. The
`TestPwModelDiagnosticCatalog.cpp:202` doc-vs-source reconciliation is the shape to copy if the
numbers must stay in code.

## History
- `#1-five-hand-copies-mesh-half-stale` `OPEN` reporter — The mesh→asset triangle/vertex count pairs for `gothic_window`, `watchtower` and `spur_gear` are hand-duplicated in five places: the normative table at `Docs/pwmodel-format.md:1323-1324` and four C++ comment blocks — `GeometryAssetCreate.cpp:415-419`, `ModelCompileHandler.cpp:438-440`, `PwModelCompiler.h:84-88`, `TestPwModelCompiler.cpp:787-790`. The report said six; an exhaustive grep of the six numerals returns exactly five files, the miscount most likely coming from `pwmodel-format.md` contributing two table rows. All five agree with each other byte-for-byte, so drift could not surface from inter-copy comparison — but the **mesh** half of every triangle pair is contradicted without needing a compile by `Docs/wiki-src/model.examples.md:13`, which declares its column to be `meshTriangleCount` from `model.validate` measured 2026-08-20: gothic_window 4,362 vs the claimed 8,174 (1.9x), watchtower 4,572 vs 10,840 (2.4x), spur_gear 15,528 vs 167,114 (10.8x). Three post-wave sources corroborate independently (`Examples/pwmodel/gothic_window.pwmodel:350`, `Examples/pwmodel/spur_gear.pwmodel:185`, `Docs/wiki-src/model.examples.op-coverage.md:88`). The asset half and all three vertex pairs cannot be re-derived — `model.validate` omits `assetTriangleCount`/`assetVertexCount` by design (`pwmodel-format.md:1330-1332`, `model.md:76`) — so those need one `model.compile` per example on post-wave binaries; no compile was attempted here. Drift cause: hand duplication plus example rework in `2988dead`, `0f2f3570`, `37fd826c` with none of the five revisited. Same family, same root cause, also stale: "6 degenerates in gothic_window, 70 in watchtower" at `PwModelCompiler.cpp:2389` and `TestPwModelCompiler.cpp:2337`, against a corpus now reporting 0 for both. Fix: re-measure, then keep one normative table and have the four comment blocks cite it by path rather than restate numbers.
