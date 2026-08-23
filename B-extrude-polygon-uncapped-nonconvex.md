---
id: B-extrude-polygon-uncapped-nonconvex
title: "append_simple_extrude_polygon reports success but does not cap a non-convex polygon (hollow mesh)"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry-script, upstream-engine, silent-false-success, hollow-mesh]
encounters: 1
lastSeen: 2026-08-15T17:21:12+05:00
---

# append_simple_extrude_polygon reports success but does not cap a non-convex polygon (hollow mesh)

Extruding a strongly non-convex outline with `bCapped = true` produces **walls with no
top**. The verb reports success, the mesh has a plausible triangle count, and the result
renders as a solid until something is seen through it. This is a silent geometric
false-success with a plausible-looking artifact — the defect class this project has spent
considerable effort eliminating.

## Reproduction — an enclosed-volume comparison, no visual needed

A spiral blade outline authored as **one 46-gon** and extruded through
`append_simple_extrude_polygon` against **the identical outline built as bands**:

| Build | Enclosed volume |
|---|---|
| One 46-gon through `append_simple_extrude_polygon` | **234977** |
| Same outline as bands | **711306** |

**3.03x less.** The shortfall is the missing cap: the mesh is a set of thin curved walls,
so the emissive plate underneath showed through the entire disc instead of only through
the joints.

An enclosed-volume check separates the two cases cleanly and needs no render. An
open-boundary check separates them just as well and is cheaper — a correctly capped
extrude is closed, this one is not.

## The fault is in the engine, not in PinWright's wrapping

Confirmed by reading engine source; no build required. The call chain:

1. `UGeometryScriptLibrary_MeshPrimitiveFunctions::AppendSimpleExtrudePolygon`
   (`Engine/Plugins/Runtime/GeometryScripting/Source/GeometryScriptingCore/Private/MeshPrimitiveFunctions.cpp:986-1032`)
   loads the caller's vertices into an `FGeneralizedCylinderGenerator` and sets
   `bCapped` (`:1025`). It performs **no validity check on the polygon beyond
   `Num() >= 3`** (`:1002-1006`) — not convexity, not simplicity, not self-intersection.
2. `FGeneralizedCylinderGenerator` defaults to `CapType = ECapType::FlatTriangulation`
   (`Engine/Source/Runtime/GeometryCore/Public/Generators/SweepGenerator.h:155`), and
   `AppendSimpleExtrudePolygon` never overrides it.
3. `FSweepGeneratorBase::ConstructMeshTopology` caps by ear-clipping —
   `PolygonTriangulation::TriangulateSimplePolygon(CrossSection.GetVertices(), OutTriangles)`
   (`Engine/Source/Runtime/GeometryCore/Private/Generators/SweepGenerator.cpp:178`) —
   into a buffer pre-sized to exactly `XVerts - 2` triangles (`:143`).
4. That ear clipper
   (`Engine/Source/Runtime/GeometryCore/Private/CompGeom/PolygonTriangulation.cpp:91-152`)
   has an explicit degenerate fallback: *"if we've tried every possible candidate vertex
   looking for an ear, go ahead and just treat the current vertex as an ear"* (`:95-98`).
   It **forces** a non-ear triangle rather than failing, always emits exactly N-2
   triangles, and reports nothing.

So the cap is emitted with the full expected triangle count — it is just geometrically
wrong (triangles outside the outline, overlapping and self-cancelling) rather than
absent. Nothing in the chain writes to the `UGeometryScriptDebug` output, which is why
the caller sees success. `TriangulateSimplePolygon` is documented as taking a *simple*
polygon and neither the generator nor the Geometry Script wrapper validates that
precondition.

Not verified without a build: which of the two unhandled cases fired for this specific
46-gon — the forced-ear fallback on a valid-but-strongly-non-convex outline, or a
self-intersecting outline (a spiral at that sweep can produce one). Both are unguarded
and both land in the same silent path, so the ticket does not depend on distinguishing
them. It is upstream in the engine's Geometry Script either way; PinWright is not in the
call path.

## PinWright's exposure is latent, not live

The same generator and the same capping path are reached by
`AppendSweepPolygon` (`MeshPrimitiveFunctions.cpp:1127-1156`, also
`FGeneralizedCylinderGenerator`, also the `FlatTriangulation` default), which
`geometry.loft`, `geometry.sweep` and `geometry.extrude_along_spline` use via
`AppendSweepPolygonCompat` (`AdvancedMeshOpsHandler.cpp:99-122`). No PinWright verb
currently trips it:

- `geometry.create_ramp` calls `AppendSimpleExtrudePolygon` with a hardcoded 3-point
  triangle (`PrimitiveHandler.cpp:710-717`) — always convex.
- `geometry.loft` / `geometry.sweep` synthesise their cross-sections as circles or
  bounding-box rectangles — always convex.
- `geometry.revolve` is the only verb that accepts a caller-supplied polygon
  (`profile`, `PrimitiveHandler.cpp:743`) and it goes to `AppendRevolvePath`, a
  different generator.

The live exposure today is `python.execute` callers driving Geometry Script directly,
which is the documented mesh-authoring route for this project. It becomes a live PinWright
defect the moment any verb accepts a caller-supplied cross-section for loft/sweep/extrude.

Origin: found during a map-authoring session,
`Docs/map/modern_landmarks.md` (Twin Gates section, "Two engine bugs, both of which
silently produce a plausible-looking wrong asset").

severity rationale: impact=silent false-success producing a geometrically wrong asset that looks correct (caller trusts it, bakes it, ships a hollow mesh) -> High; reach=reachable through `python.execute`, the documented Geometry Script authoring route for this project, and latent in three PinWright verbs -> no modifier -> High.

**Workaround:** Do not extrude a strongly non-convex outline in one call. Decompose it
into bands or partial revolves and extrude each. Note that straight quads are not a free
substitute — the second attempt on this blade, 22 straight quads, still failed a numeric
coverage test of the ideal sector at **94.3%**, because a quad chords the arc it should
follow (sagitta 40 uu for a 48-degree span at r 464) and the shortfall sits exactly on
the band edges, where it reads as radial striping. The shipped blade is **48 partial
revolves**, which follow the true arc.

**Fix:** Two parts, independent.

1. *Upstream.* Report to Epic: `AppendSimpleExtrudePolygon` (and `AppendSweepPolygon`)
   should either validate the cross-section before capping, or propagate a
   `UGeometryScriptDebug` error when the flat triangulation cannot produce a valid cap,
   instead of relying on the ear clipper's forced-ear fallback and returning a mesh whose
   cap is silently wrong.
2. *Here — the guard worth having regardless of upstream.* Add a post-condition check
   after any capped extrude/sweep: assert the result is closed (open boundary edges == 0)
   or that its enclosed volume is within tolerance of the expected value. PinWright
   already computes both signals — `geometry.check_health` returns
   `{isClosed, boundaryEdges, ...}` and `geometry.measure` returns volume
   (`MeshMeasureHandler.cpp`) — so the check is a call, not new maths. Fold it into the
   capped paths of `geometry.create_ramp` / `loft` / `sweep` / `extrude_along_spline` and
   fail loud with the boundary-edge count rather than returning a hollow mesh as success.
   Document the trap on the `geometry.extrude` / `geometry.create_ramp` /
   `geometry.extrude_along_spline` method pages (`docs/wiki-src/geometry.md`): Geometry
   Script's flat-triangulation cap is only valid for a simple, near-convex cross-section;
   verify with `geometry.check_health` before baking.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a map-authoring session (`Docs/map/modern_landmarks.md`, Twin Gates). Repro on UE 5.8 via `python.execute`: extrude a 46-vertex spiral blade outline through `append_simple_extrude_polygon` with `b_capped=True`; the call succeeds and the mesh encloses a volume of **234977** against **711306** for the identical outline built as bands — 3.03x less — and it renders as thin curved walls with no top. An enclosed-volume or open-boundary check distinguishes the two cases numerically with no visual. Root cause confirmed in engine source without a build: `AppendSimpleExtrudePolygon` (`MeshPrimitiveFunctions.cpp:986-1032`) validates only `PolygonVertices.Num() >= 3`, uses `FGeneralizedCylinderGenerator` whose `CapType` defaults to `ECapType::FlatTriangulation` (`SweepGenerator.h:155`), which caps via `PolygonTriangulation::TriangulateSimplePolygon` (`SweepGenerator.cpp:178`) into a buffer pre-sized to exactly `XVerts - 2` triangles (`:143`); that ear clipper has an explicit "just treat the current vertex as an ear" degenerate fallback (`PolygonTriangulation.cpp:95-98`) which forces non-ear triangles rather than failing and never reports an error, so the cap is emitted at full triangle count but geometrically wrong. Fault is upstream in the engine's Geometry Script, not in PinWright's wrapping. Not determined without a build: whether the specific 46-gon hit the forced-ear fallback or was self-intersecting — both are unguarded and land in the same silent path. PinWright's exposure is latent: the same capping path backs `AppendSweepPolygon` used by `geometry.loft` / `sweep` / `extrude_along_spline`, but every PinWright cross-section is currently synthesised convex.
- `#2-additional-current-source-scope` `OPEN` reporter — Additional evidence: **Adversarial review A — upstream validation gap remains, but the reported PinWright/non-convex hollow-cap framing is only partial.** Actuality: PARTIAL. Framing: UE 5.8 `AppendSimpleExtrudePolygon` and `AppendSweepPolygon` still reject only fewer than three points (`C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Private\MeshPrimitiveFunctions.cpp:986-1027`, `:1096-1156`), then use flat triangulation with a forced-ear fallback that never reports failure (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\Generators\SweepGenerator.cpp:141-199`, `C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\CompGeom\PolygonTriangulation.cpp:95-168`). However, the triangulator explicitly handles concave ears (`:104-129`), so “strongly non-convex” alone does not establish a defect; the old 46-gon repro did not establish whether it was simple-concave or self-intersecting. Current typed PinWright paths synthesize circles/rectangles or a hardcoded triangle (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\AdvancedMeshOpsHandler.cpp:363-447`, `:519-539`, `:866-881`; `PrimitiveHandler.cpp:710-717`), while `python.execute` is an arbitrary escape hatch (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\System\PythonExecuteHandler.cpp:63-194`). Proposed fix: **INCOMPLETE**, because `geometry.check_health` only checks boundary/degenerate/bowtie topology (`MeshMeasureHandler.cpp:167-223`), so a wrong but topologically closed cap can pass; volume has no expected reference, and `extrude_along_spline` passes `bLoop=true`, where the engine intentionally ignores caps (`AdvancedMeshOpsHandler.cpp:906-910`; `SweepGenerator.cpp:603-612`). Runtime: NOT VERIFIED in this review; only the ticket’s historical #1 numbers exist. Recommendation: **REFRAME**; keep the upstream issue as “invalid/non-simple polygon cap silently accepted,” add a current UE 5.8 differential repro for known simple-concave versus self-intersecting inputs, and validate triangulation geometry (not boundary count) before choosing an Epic report or typed-handler guard.
- `#3-additional-cap-contract` `OPEN` reporter — Additional evidence: **Adversarial review B — A’s PARTIAL/REFRAME conclusion is supported; I disagree that a boundary post-check is a meaningful guard.** Actuality: PARTIAL. Framing: UE documents `TriangulateSimplePolygon` for simple polygons (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Public\CompGeom\PolygonTriangulation.h:16-24`), while Geometry Script only checks `Num() >= 3` and feeds the default flat triangulation (`C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Private\MeshPrimitiveFunctions.cpp:1002-1027`, `:1115-1144`; `C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Public\Generators\SweepGenerator.h:141-155`). The ear test explicitly handles concave ears (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\CompGeom\PolygonTriangulation.cpp:104-129`), so “strongly non-convex” is not itself a defect; only the forced-ear/undefined path for non-simple or degenerate inputs is established. The 46-gon coordinates and source map named by the ticket are absent in this checkout, so the “uncapped/open boundary” symptom is NOT VERIFIED. Flat caps reserve N-2 triangles and reuse end-section vertices (`C:\UE_5.8\Engine\Source\Runtime\GeometryCore\Private\Generators\SweepGenerator.cpp:141-186`), meaning a geometrically wrong cap can still be topologically closed; `geometry.check_health` cannot catch it (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\MeshMeasureHandler.cpp:167-223`). Current typed loft/sweep paths synthesize convex circles/rectangles, and `extrude_along_spline` sets `bLoop=true` (caps ignored) (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\AdvancedMeshOpsHandler.cpp:363-447`, `:519-617`, `:866-910`); arbitrary `python.execute` is the only current escape hatch (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\System\PythonExecuteHandler.cpp:63-194`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\level-building.md:113`). Proposed fix: **INCOMPLETE**, because boundary/volume post-checks without an expected reference are a band-aid; systemic work is input validation plus an explicit engine/debug failure, with a deterministic simple-concave versus self-intersection regression. High severity is overstated for a non-normal, escape-hatch-only PinWright path; the board reach rule points toward Medium. Runtime: NOT VERIFIED (no build/run permitted; only historical numbers). Recommendation: **REFRAME**; retain as an upstream/API-contract gap, add the actual polygon and current UE 5.8 repro, then decide engine report versus typed guard.

- `#4-guard-at-the-input` `IN-REVIEW` developer — Chose **reject at the input**, and both reviews' objections drove that choice rather than being worked around. There is no parameter to implement: the fault is upstream (AppendSimpleExtrudePolygon validates only Num()>=3, caps by TriangulateSimplePolygon whose ear clipper forces a non-ear rather than failing, and writes nothing to the debug channel). Review A and B are right that the ticket's proposed post-condition is not a guard — the cap reuses the end-section vertices so a geometrically wrong cap is still topologically CLOSED, `check_health` reads clean, and enclosed volume has no reference. The input is the only place the defect is detectable, so that is where the check went. Added `GeometryOps::PolygonSignedArea` and `GeometryOps::PolygonIsSimple` (GeometryOps_Primitives.h/.cpp): point count, coincident consecutive points, enclosed area, and non-adjacent edge crossings, with the offending indices named in the reason. Both reviews also warned that 'strongly non-convex' is not itself a defect — the ear test handles concave ears — so the predicate ACCEPTS concave outlines and the test asserts that with a concave L and a deep-reflex arrowhead as known-good controls beside a bowtie, a collinear run and a repeated point as known-bad. **The latent exposure turned out to have a live instance.** `geometry.create_ramp` is the only verb driving AppendSimpleExtrudePolygon; its wedge is synthesised but its shape comes from caller `length`/`height`, so `height=0` collapsed the outline onto a line and returned a successful mesh with a degenerate cap, and a negative value mirrored the wedge and capped it inside-out. Both are now INVALID_ARGUMENT naming the numbers. `geometry.revolve`'s caller-supplied profile is deliberately untouched: AppendRevolvePath is a different generator and is not on the flat-triangulation cap path. Upstream half of the fix (an Epic report) NOT done. Tests: Tests/Geometry/TestGeometryCappedCrossSectionGuard.cpp, ids `.cross_section.SimplicityAcceptsConcaveAndRejectsSelfIntersecting` and `.create_ramp.DegenerateWedgeIsRefusedNotSilentlyCapped`. Docs/wiki-src/geometry.md gains a `### geometry.create_ramp` section explaining the cap trap. Commits 43e2e2b6, e4e631c0, 4f7110e3. Compile-checked -SingleFile, Result: Succeeded. Vision verification pending a link window — a peer's suite currently holds the DLLs.
