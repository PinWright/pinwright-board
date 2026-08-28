---
id: B-pwmodel-health-no-self-intersection
title: "The documented `.pwmodel` health gate `isClosed && signedVolume > 0` returns green on a solid whose walls have been pushed through each other — `health` carries no self-intersection field, and `signedVolume` degrades smoothly with no threshold, so it only catches the pinch after the sign has already flipped"
status: DONE
severity: Medium
category: bug
tags: [pwmodel, health, signedVolume, self-intersection, model.validate, model.compile, sweep, false-green, missing-signal]
encounters: 2
lastSeen: 2026-08-28T08:30:00+05:00
---

# A closed mesh that is no longer a solid reports every health field healthy

A closed, consistently-oriented, degenerate-free mesh whose two walls have been pushed **through**
each other reports **every** `health` field healthy. `isClosed: true`, `boundaryEdges: 0`,
`orientationConsistent: true`, `degenerateTriangles: 0`, `nonManifoldVertices: 0`. The only field
that moves at all is `signedVolume`, and it moves **smoothly** — there is no threshold at which the
response says anything is wrong. It quietly stops being the volume of the solid the author wrote,
and eventually changes sign.

The documented gate is `health.isClosed && health.signedVolume > 0` (`model.compile` wiki,
§ *`health.signedVolume` is the inside-out signal*). That gate **passes** on meshes that are
visibly not solids.

## Root cause — NOT traced into plugin source; this is a missing signal, not a wrong line

**This defect was measured through the `model.validate` response surface and has not been traced to
a guilty line, because there is no wrong line to cite — the check does not exist.** What is
source-verified is the *absence*, which is a checkable fact rather than a hypothesis:

- The health block emitted by `model.validate` / `model.compile` is
  `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Model/ModelCompileHandler.cpp:647-669`.
  It emits exactly `isClosed`, `boundaryEdges`, `degenerateTriangles`, `nonManifoldVertices`,
  `signedVolume`, `orientationConsistent` — and per-part duplicates at `:815-817`. There is no
  self-intersection field and no code path that could produce one.
- A case-insensitive `selfintersect` sweep over all of `Source/` returns only test helpers
  (`TestGeometryModelingOpOptions.cpp:167`, `:1465`) and a 2D-polygon guard — nothing on the mesh
  health path.

Two facts about the surrounding tree sharpen the gap, and both are verified:

- **The format already refuses self-intersection in 2D.** `PolygonIsSimple`
  (`Handlers/Geometry/GeometryOps_Primitives.cpp:711-760`) rejects a profile outline with a named
  reason — *"edge %d-%d crosses edge %d-%d, so the outline is self-intersecting"*. The concept is
  first-class in the vocabulary for the *input* outline and absent for the *output* solid.
- **The plugin already warns an author about this exact failure mode prospectively, and can never
  confirm it.** The `spherify` / `cylindrify` anisotropy warning
  (`Handlers/Geometry/GeometryOps_Modeling.cpp:2263-2272`) tells the reader in prose that *"a
  displacement op run afterwards (noise_deform, displace, the warp deformers) self-intersects
  wherever its magnitude approaches the local edge length, which is what shows up as a black
  cavity"*. The author is warned that self-intersection is the thing to fear, and then handed a
  `health` block that cannot measure it.

## Verbatim repro — the twist sweep

Sweep a thin rectangular section along a straight path with the section rotating (a twisted ribbon),
emitted through `append_buffers` as a closed 4-sided tube. Straight **200 x 10 x 1800** test tube,
**15 rings**, `model.validate` on each. Measured 2026-08-27, UE 5.8, this checkout, while building
`SM_Kelp_Blade`:

| total twist | `signedVolume` | analytic swept volume | isClosed | boundaryEdges | orientationConsistent | degenerateTriangles |
|---|---|---|---|---|---|---|
| 0   |  3,600,000 | 3,600,000 | true | 0 | true | 0 |
| 90  |  2,245,522 | 3,600,000 | true | 0 | true | 0 |
| 180 |    892,987 | 3,600,000 | true | 0 | true | 0 |
| 310 | -1,022,818 | 3,600,000 | true | 0 | true | 0 |

At **61 rings** the same 310-degree case gives **2,511,785** — converging with tessellation, but
still 30% short.

Read the table as a gate: the published `isClosed && signedVolume > 0` **passes** the 90 and
180 degree rows, which are 62% and 25% of the volume they should have. Only the 310 row fails, and
only because the sign has flipped by then — the mesh has been wrong for a long way before the one
signal fires.

An author with no independent analytic volume has nothing to compare `signedVolume` to. It is a bare
number with no expected value, and no other field moves at all.

## Mechanism of the pinch (author-side, for the repro to be reproducible)

Connecting corner *j* of one ring to corner *j* of the next through a rotation sags the ruled quad
at its centre by about `half_width * dtheta / 2`, and the budget for that sag is the section's
**half thickness**. At 105 wide against 5.5 thick, any visible twist collapses the section at every
quad centre and then pushes the top wall through the bottom one. This is why the failure is
smooth: each quad crosses a little further, and `signedVolume` integrates the crossing continuously.

## What it should do

Either, and the second is the cheap one:

- A **self-intersection count** in `health`. The engine API is present in UE 5.8:
  `TMeshAABBTree3::TestIntersection` and `FindAllIntersections`
  (`Source/Runtime/GeometryCore/Public/Spatial/MeshAABBTree3.h:947`, `:1060`), instantiated as
  `FDynamicMeshAABBTree3` (`DynamicMesh/DynamicMeshAABBTree3.h:13`) — and the collision path already
  builds trees.
- Or, cheaper and probably enough for the gate: **warn when a closed part's `signedVolume` is a small
  fraction of its bounding-box volume.** The pinched cases here are 4-25% of a box the mesh visibly
  fills, and `bounds` is already computed on the same mesh at the same stage
  (`Model/PwModelCompiler.cpp:3303`), so the ratio costs nothing.

## Workaround

Compute the expected swept volume in the producer and compare against `signedVolume` by hand. The
kelp shipped as an **open double-sided sheet** instead, which has no interior to pinch. Both
workarounds are outside the format: one needs arithmetic the format cannot express, the other
abandons the solid.

## Distinct from related tickets

- **This contradicts a design premise standing in `F-sweep-per-frame-scale-law`** (IN-REVIEW,
  Medium). That ticket argues at `:51-53` that a mesh which "renders correctly and lights
  inside-out" is *"exactly the class of defect `health.signedVolume` exists to catch after the
  fact"*, and leans on that backstop to justify **holding** rather than extrapolating a `scales`
  curve. The refusal design is still right — but the backstop is weaker than the ticket claims.
  `signedVolume` catches a *uniform* sign flip; it does not catch a partial pinch, and the table
  above shows it degrading smoothly through 38% and 75% error while every other field stays green.
  An encounter section recording this has been appended to that ticket; its text is otherwise
  untouched.
- `B-pwmodel-overlap-check-skips-modifiers` (OPEN, Medium, encounters 2) is **adjacent but not the
  same defect** — cross-link, do not dedup. Its `:36` says the four append-style ops "hand the next
  boolean a self-intersecting mesh", which is the same words for a different fault: that ticket is
  about `PWMODEL_UNUNIONED_OVERLAP` being wired to the wrong call site, i.e. a diagnostic for
  overlap **between parts/copies** that cannot fire on `mirror` or the three `array_*` ops. This
  ticket is about a **single part** self-intersecting with itself, for which no diagnostic exists at
  any call site. Fixing either leaves the other standing.
- No board ticket mentions the `.pwmodel` compile/validate **response** shape at all — `bounds`
  appears in zero tickets, and this is the first ticket on `health`'s completeness.

severity rationale: impact=soft blocker — every field the response emits is TRUE of the mesh as built (`signedVolume` really is that mesh's signed volume), so nothing lies; the defect is an absent signal on the surface the docs name as the correctness gate, and the workaround (compute the intended solid's analytic volume in the producer) exists but lives outside the format × reach=`health` runs on every validate/compile, but the pinch class is confined to closed parts whose section can self-cross (sweeps, tubes, twisted ribbons) rather than to every document, so no reach bump -> Medium. Rated to match `B-pwmodel-overlap-check-skips-modifiers`, the sibling missing-diagnostic ticket on the same surface. Not High: no field returns wrong or stale data, and the docs do not claim `health` is exhaustive.

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD, while building `SM_Kelp_Blade`. Twist sweep, 200 x 10 x 1800 straight tube emitted through `append_buffers` as a closed 4-sided tube, 15 rings, `model.validate` per twist: 0deg `signedVolume` 3,600,000 (= analytic); 90deg **2,245,522**; 180deg **892,987**; 310deg **-1,022,818** — with `isClosed: true`, `boundaryEdges: 0`, `orientationConsistent: true`, `degenerateTriangles: 0` on EVERY row. At 61 rings the 310deg case gives 2,511,785, converging with tessellation but still 30% short. The published gate `health.isClosed && health.signedVolume > 0` (`model.compile` wiki § "`health.signedVolume` is the inside-out signal") therefore PASSES the 90 and 180 rows, which are 62% and 25% short of the solid the author wrote; only the sign flip at 310 fails, by which point the mesh has been wrong for a long way. Author-side mechanism: connecting corner j of one ring to corner j of the next through a rotation sags the ruled quad by about `half_width * dtheta / 2` against a budget of the section's HALF THICKNESS — 105 wide against 5.5 thick collapses at every quad centre and pushes the top wall through the bottom. **Root cause NOT traced into plugin source, and deliberately not guessed: there is no wrong line, the check does not exist.** What IS source-verified at HEAD is the absence and its surroundings — the health block is emitted at `ModelCompileHandler.cpp:647-669` (plus per-part at `:815-817`) and carries only `isClosed`/`boundaryEdges`/`degenerateTriangles`/`nonManifoldVertices`/`signedVolume`/`orientationConsistent`; a `selfintersect` sweep over all of `Source/` finds only test helpers; the format ALREADY refuses a self-intersecting 2D outline by name (`PolygonIsSimple`, `GeometryOps_Primitives.cpp:711-760`); and the plugin ALREADY warns an author prospectively that a displacement op "self-intersects wherever its magnitude approaches the local edge length" (`GeometryOps_Modeling.cpp:2263-2272`) on a surface that can never confirm or deny it. Suggested fix: a self-intersection count via `TMeshAABBTree3::TestIntersection` / `FindAllIntersections` (`MeshAABBTree3.h:947`, `:1060`; `FDynamicMeshAABBTree3` typedef at `DynamicMeshAABBTree3.h:13`), or the cheap proxy — warn when a closed part's `signedVolume` is a small fraction of its bounding-box volume (4-25% here), using the `bounds` already computed on the same mesh at `PwModelCompiler.cpp:3303`. Worked around by computing the expected swept volume in the producer and shipping the kelp as an open double-sided sheet; defect untouched. Deduped: NO-MATCH board-wide (1316 tickets). Cross-links: contradicts the standing design premise in `F-sweep-per-frame-scale-law:51-53` (encounter appended there, its text untouched); adjacent to but distinct from `B-pwmodel-overlap-check-skips-modifiers`, which is about overlap BETWEEN parts being undiagnosable, not a single part pinched through itself.
- `#2-covered-by-the-shared-embedding-signal` `IN-REVIEW` developer -- Fixed by the same single
  measurement added for `B-pwmodel-health-blind-to-interior-membrane`, which that ticket asked be
  designed together with this one rather than as two checks. The shared statement is that the
  surface is not an EMBEDDED boundary; the membrane and the pinch are the coplanar and the
  transversal case of the same triangle-pair test. Took the first of this ticket's two suggested
  fixes, `TMeshAABBTree3::FindAllSelfIntersections`, and NOT the cheap `signedVolume` /
  bounding-box-volume ratio proxy: that ratio is 4-25% on a legitimate torus, lattice or L-shape
  too, so it would have fired on correct geometry and been switched off wholesale. New
  `GeometryUtils::MeasureMeshSelfIntersection` (`Handlers/Geometry/GeometryUtils.h/.cpp`): one
  `FMeshConnectedComponents` walk, one AABB tree and one self-intersection descent PER COMPONENT,
  `bIgnoreTopoConnected=true`, coplanar reporting forced on, degenerate triangles excluded via
  `FQueryOptions.TriangleFilterF`; declined above 200k triangles, pair count capped at 4096, both
  states reported rather than approximated. Counted per shell so appended interpenetrating parts
  and siblings - the normal organic-model idiom, already covered by
  `PWMODEL_UNUNIONED_OVERLAP{,_PARTS}` - stay at zero. Published as
  `health.selfIntersections` / `selfIntersectingComponents` / `selfIntersectionsTruncated` on
  `model.compile` and `model.validate`, warned as `PWMODEL_SELF_INTERSECTING_SURFACE` with a
  witness coordinate, documented in `docs/pwmodel-format.md` where the gate is now
  `isClosed && signedVolume > 0 && selfIntersections === 0`. Tests:
  `Tests/Model/TestPwModelSelfIntersection.cpp` - the transversal case is
  `PinWright.Model.SelfIntersection.WallsCrossingInsideOneShellAreCounted`, a hand-built
  single-component surface whose wall passes through its own floor, plus four controls that must
  stay at zero. NOT re-measured: the twist-sweep table in this ticket was produced against real
  content and was not reproduced here; the transversal fixture is synthetic. `geometry.check_health`
  and its `healthy` verdict are deliberately NOT wired to the new measurement - out of this
  ticket's scope and a different cost class for that verb's callers.

- `#3-verified-fixed` `DONE` verifier — 2026-08-28. Plugin rebuilt from a clean tree at `b79ba53e` and verified against disk, not against the build's own success message: `UnrealEditor-PinWright.dll` 39,898,624 -> 40,644,096 bytes at 2026-08-28 08:11:48, `UnrealEditor-PinWrightGeometry.dll` 4,983,296 -> 5,113,344, canonical link with no `-000N` artifacts in `UnrealEditor.modules`. Editor restarted on that DLL and the ticket's own repro re-run. Signal exists and detects. `health` now carries `selfIntersections`, `selfIntersectingComponents` and `selfIntersectionsTruncated`. Verified with both a negative and a positive control rather than a single case: a clean closed revolve reports `selfIntersections: 0`, and an axis-crossing revolve (`profile=[(-60,0),(100,0),(100,40),(-60,40)] steps=24`) reports **3341** crossing pairs in 1 shell plus a `PWMODEL_SELF_INTERSECTING_SURFACE` warning that names the first crossing at (-53.4993, -26.2874, -0) and explicitly distinguishes itself from `PWMODEL_UNUNIONED_OVERLAP`. Every other health field stayed green on that mesh - `isClosed` true, `boundaryEdges` 0, `orientationConsistent` true - which is exactly the blindness the ticket reported.
