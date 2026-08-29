---
id: B-example-triangle-counts-stale
title: "The mesh/asset triangle-count pairs for gothic_window, watchtower and spur_gear are hand-duplicated in five places and the mesh half of every one is stale — off by up to 10.8x"
status: IN-REVIEW
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

- `#2-additional-current-drift` `OPEN` reporter — Additional evidence: **Adversarial review A — current stale values remain after the corpus rework.** Actuality: CONFIRMED CURRENT. Framing: the five-file duplication and “up to 10.8x” remain accurate for the six old numerals, but the corpus now reports `gothic_window` 2,222 and `watchtower` 2,444 mesh triangles versus the normative 8,174 and 10,840 (3.68x/4.44x), while `spur_gear` remains 15,528 versus 167,114 (10.76x); further stale prose includes `gothic_window.pwmodel:378` (4,362) and `model.examples.op-coverage.md:88` (4,312/4,572). Proposed fix: INCOMPLETE, because remeasurement plus C++ table citations reduces copies but does not enforce the table, cover extra prose, or reconcile asset/vertex counts; the existing asset-count test is synthetic and does not audit shipped examples. Evidence: `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Docs\pwmodel-format.md:1370-1371`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Docs\wiki-src\model.examples.md:13-31`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Docs\wiki-src\model.examples.op-coverage.md:88`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\Model\TestPwModelCompiler.cpp:798-843`, `C:\UE_5.8\Engine\Source\Developer\MeshBuilder\Private\StaticMeshBuilder.cpp:1644-1675`. Runtime: NOT VERIFIED. Recommendation: REFRAME; keep OPEN, run one current post-wave `model.compile` per example for mesh/asset/vertex/health/collision counts, generate/reconcile the canonical table and prose, and add a corpus drift check.
- `#3-additional-adversarial-scope` `OPEN` reporter — Additional evidence: **Adversarial review B — A’s core finding survives, but its scope and guard need correction.** Actuality: CONFIRMED CURRENT. Framing: the old mesh→asset pairs remain unqualified in the canonical table and four C++ comment blocks (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Docs\pwmodel-format.md:1370-1371`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Model\PwModelCompiler.h:90-124`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Model\ModelCompileHandler.cpp:435-456`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\GeometryAssetCreate.cpp:434-455`, `X:\src\unreal\EAContentExamples58\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\Model\TestPwModelCompiler.cpp:782-843`), while the remeasured example table says 2,222/2,444/15,528 (`X:\src\unreal\EAContentExamples58\Plugins\PinWright\Docs\wiki-src\model.examples.md:13-31`), so “up to 10.8x” still holds. I agree `gothic_window.pwmodel:378` and `model.examples.op-coverage.md:88` are extra unqualified stale claims, but `model.authoring.md:211` is explicitly historical and must not be bulk-rewritten. High is defensible for a normative reference, though this is documentation drift rather than a runtime RPC defect. Proposed fix: INCOMPLETE, because remeasurement plus table links removes duplicated comments but leaves a hand-maintained canonical table; the only count test is a synthetic box (`...TestPwModelCompiler.cpp:798-843`), and a fixed-count corpus assertion could fail legitimately across UE versions. Runtime: NOT VERIFIED. Recommendation: REFRAME; keep OPEN, remeasure the three examples’ four counts and health, reconcile every unqualified claim, remove numeric copies, and add a version-aware/generated catalog check rather than a brittle exact-count test.
- `#4-duplication-collapsed-numbers-withdrawn-guard-added` `IN-REVIEW` developer — Collapsed the five hand-copies to ONE normative statement and withdrew every number I could not verify in this checkout today. **Measurement was not possible:** `mcp__pinwright__call` returned `EDITOR_NOT_RUNNING` (connection refused) and the ticket forbids starting an editor, so no `model.compile` was run and no asset-side or vertex count was re-derived. Entries #2 and #3 were written against the EAContentExamples58 checkout and no figure from either was copied. **(A) Duplication collapsed.** `Docs/pwmodel-format.md` (section "Four counts: two for the mesh, two for the asset", now ~line 1758) keeps the mechanism/direction table and states itself as the single normative source; the four C++ comment blocks — `GeometryAssetCreate.cpp`, `ModelCompileHandler.cpp`, `PwModelCompiler.h`, `Tests/Model/TestPwModelCompiler.cpp` — now cite it by path and section title and restate no numbers. **(B) Numbers.** The `Measured` column (three mesh→asset triangle pairs, three vertex pairs) was deleted rather than refreshed, with a dated withdrawal note saying why and routing readers to the `Tris` column of `Docs/wiki-src/model.examples.md` for mesh counts and to a live `model.compile` for asset counts. Same-family staleness turned out to be wider than the ticket listed: the "6 in gothic_window, 70 in watchtower" degenerate claim was in FIVE files (`GeometryUtils.h`, `PwModelCompiler.cpp`, `PwModelCompiler.h`, `PwModelDiagnostic.h`, `TestPwModelCompiler.cpp`), not two. The counts are removed from all five, and `PwModelDiagnostic.h`'s present-tense "two examples ship with them" is corrected — `model.examples.md` now reports 0 degenerates for every example. Prose: `Examples/pwmodel/gothic_window.pwmodel` loses "the real 4,362 triangles" and `model.examples.op-coverage.md:88` loses "now compiles 4,572 triangles" (its 4,312 / 64 figure is the record of a past A/B experiment and is left standing as history). **(D) Left alone as instructed:** `Docs/wiki-src/model.authoring.md` — the 54 → 144 vertex jump, explicitly dated 2026-08-19 — is untouched. **(C) Drift guard**, new, following the `TestPwModelDiagnosticCatalog.cpp` doc-vs-source shape and placed beside it: `Source/PinWright/Private/Tests/Core/TestPwModelExampleCatalog.cpp`, ids `PinWright.core.pwmodel_examples.ShippedFilesMatchThePublishedTable` and `PinWright.core.pwmodel_examples.SourceCitesTheCountTableRatherThanCopyingIt`. Deliberately NOT an exact-count assertion — both reviewers warned a pinned count fails legitimately across UE versions, and it would itself become a sixth hand-copy. It asserts three version-independent things: the shipped `Examples/pwmodel/*` roster and the published corpus table name the same set (skeletal examples, discovered by a `use skeleton` line rather than listed, are exempt from the table but must still be named in `pwmodel-format.md`); the heading the four comments cite still exists under that title; and no line under `Source/` pairs a shipped example's name with a count again (two regex shapes covering both withdrawn forms, with the guard file excluding itself so its own worked counter-examples do not self-trip). Verified by re-implementing all three checks as a script against this tree: 14 shipped files, 1 skeletal, 13 expected rows against 13 table rows, no missing and no phantom rows; cited heading present; zero count-pair hits anywhere under `Source/` outside the guard file. `Content/Python/check_test_ids.py` → CLEAN, 4766 ids, no dot-prefix collisions, no duplicates. **NOT DONE:** the plugin was not compiled and the new test was not executed (both forbidden for this task), and the asset-side and vertex counts remain unmeasured — they are now absent rather than wrong, and re-deriving them needs one `model.compile` per example against a live editor.
