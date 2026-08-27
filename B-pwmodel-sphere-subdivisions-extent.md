---
id: B-pwmodel-sphere-subdivisions-extent
title: "`sphere subdivisions=2` builds a shape whose half-extent is `radius/sqrt(3)`, 42% short of the `radius=` it was handed, and neither `describe_ops` nor the warning that already fires at that floor says so — geometry placed against `radius=` lands outside the mesh"
status: OPEN
severity: Medium
category: bug
tags: [pwmodel, sphere, subdivisions, AppendSphereBox, extent, bounds, silent-trap, floatingGeometry, docs-gap]
encounters: 1
lastSeen: 2026-08-27T18:57:03+05:00
---

# At its minimum subdivision, `sphere` is 42% smaller than its own `radius=` reads

`sphere subdivisions=2` is the cheap 12-triangle primitive — that is what the floor is for. At
that setting the shape's half-extent on every axis is **`radius / sqrt(3)` = 0.5774 * radius**,
not `radius`. Nothing in the response says so. `bounds` reports the true (small) box, so no field
lies; the trap is that `radius=` is the only number the author wrote and it is not the number the
shape measures.

An author who reads `radius=` as a half-extent — the ordinary reading, and the one every other
primitive in the vocabulary supports — and then places geometry against it puts that geometry
**outside** the mesh, by 42% of the radius on each axis.

## Root cause (guilty source lines)

The plugin side is a **missing warning at a site that already warns**:
`Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Primitives.cpp:215-243`
(`GenerateSphere`). Two clamps run there, and the second exists purely to report the floor:

```cpp
    Params.Subdivisions = ClampSegmentsWarn(Params.Subdivisions, 16, TEXT("subdivisions"), Result);   // :220

    // AppendSphereBox's real floor is 2, not ClampSegments' 1. ...
    Params.Subdivisions = ClampRangeWarn(Params.Subdivisions, 2, GEOM_MAX_SEGMENTS,                   // :227
        TEXT("subdivisions"), Result);
```

So `subdivisions=1` gets a warning naming the floor, and `subdivisions=2` — the value the clamp
raises it to, and the one that carries the extent trap — gets nothing. The long comment at `:232-237`
documents the polyhedron and the triangle counts (`12*(N-1)^2`, measured `N=2 -> 12`) and says
nothing about the extent that follows from them.

The geometry is engine-side and is verifiable, not inferred:

- `AppendSphereBox` (`MeshPrimitiveFunctions.cpp:379-382`, UE 5.8) sets
  `SphereGenerator.EdgeVertices = FIndex3i(StepsX, StepsY, StepsZ)`. `EdgeVertices` is **vertices
  per cube edge**, so `subdivisions=2` gives 2 vertices per edge — one quad per face, and the only
  vertices in the mesh are the **8 cube corners**. (The plugin's own measured count, 12 triangles
  at `N=2`, is the same fact seen from the triangle side.)
- `FBoxSphereGenerator::Generate` (`Generators/BoxSphereGenerator.h:39-70`) projects each box vertex
  onto the sphere. For a corner, normalized box coordinates are `(±1, ±1, ±1)`, so the cube-map term
  at `:53` is `sx = 1 * sqrt(1 - 0.5 - 0.5 + 1/3) = sqrt(1/3)`, likewise `sy`, `sz`; `Normalize(V)`
  leaves that vector unchanged (its length is already 1); `:69` scales by `Radius`. Every corner
  therefore lands at `(±r/sqrt(3), ±r/sqrt(3), ±r/sqrt(3))`. Both projection methods agree here —
  `NormalizedVector` gives `(1,1,1)/sqrt(3)`, the same point.

From `subdivisions=3` up there are edge- and face-centre vertices that do reach `r` on an axis, the
bounding box matches `radius=`, and the trap disappears. It is confined to the floor.

## Verbatim repro

Two documents, `model.validate` on each, `bounds` read off the response — measured 2026-08-27,
UE 5.8, this checkout:

```
part p { sphere radius=100 subdivisions=2 }   ->  bounds size 115.5   (half-extent 57.7)
part p { sphere radius=100 subdivisions=3 }   ->  bounds size 200     (half-extent 100)
```

`115.47 = 2 * 100 / sqrt(3)`.

The instance that cost real work, on `SM_Fish_A` during this build — an anisotropically scaled
sphere, so all three axes show the factor independently:

```
sphere radius=1 subdivisions=2 scale=(4.4, 1.3, 2.5)
  ->  half-extents (2.54, 0.75, 1.44)
      expected     (4.40, 1.30, 2.50)
      4.4/sqrt(3)=2.540   1.3/sqrt(3)=0.751   2.5/sqrt(3)=1.443
```

The caudal fin was rooted against the *nominal* peduncle radius, which put the root **0.5664997 uu
outside** the peduncle it was meant to be embedded in. The compile was otherwise green — `success:
true`, no error, no warning naming the size — and the **only** field that reported it was
`floatingGeometry`: `floatingCount: 2`, `nearestDistance: 0.5664997`. Without that field the fish
would have shipped with a detached tail.

## What it should do

Either is sufficient, and (b) is the cheaper:

- **(a)** Warn at `subdivisions=2` naming the factor, at the site that already warns about the floor
  (`GeometryOps_Primitives.cpp:227`) — the same shape as the clamp warnings beside it.
- **(b)** Put the factor in the `subdivisions` parameter text in `model.describe_ops`
  (`Model/PwModelParser.cpp:468`), which today reads *"Quad steps per cube face … Triangle count is
  12*(subdivisions-1)^2. Minimum 2: 1 draws the same 12-triangle cube as 2 and is clamped up with a
  warning."* — one clause short of the thing an author needs.

## Workaround

Treat the `subdivisions=2` half-extent as `radius / sqrt(3)` (written into `SM_Fish_A.pwmodel`'s
header so the next reader of that file does not re-derive it), or use `box` when a box is what is
wanted. Fin roots were moved inside the *measured* box; defect untouched.

## Distinct from related tickets

- `B-blossom-pole-ring-degenerate-triangles` (IN-REVIEW, Medium) is the only adjacent ticket, and it
  cites this same construction only to reassure: its `#2-fixed` entry says *"`sphere` is a pole-free
  box-sphere so the defect class cannot recur"* when scoping a `.pwmodel` re-authoring of the
  blossom clusters. **That statement is correct and this ticket narrows it**: the collapsed-pole
  defect class genuinely cannot recur on `AppendSphereBox`, but the same construction carries a
  *different*, unreported trap at its minimum subdivision. Anyone porting a UV-sphere recipe to
  `sphere` on the strength of that citation walks into this one. That ticket never mentions extent,
  `radius=`, `subdivisions=` or `bounds`.
- Nothing else on the board mentions `floatingGeometry`, `floatingCount` or `nearestDistance` at
  all — this is the first ticket on the only field that caught the defect.

severity rationale: impact=soft blocker — every reported field is true of the mesh as built (`bounds` reports the real 115.5 box), so nothing lies; the gap is that the parameter an author writes is not the extent it produces, and the remedy required deriving the factor from engine source × reach=`subdivisions=2` is a specific setting rather than the default (16), so no reach bump -> Medium. Not High: no field returns wrong or stale data. Not Low: the consequence is misplaced geometry on a green compile, caught here only by `floatingGeometry`.

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Two-document repro: `sphere radius=100 subdivisions=2` reports `bounds` size **115.5**, `subdivisions=3` reports **200** — `115.47 = 2*100/sqrt(3)`. Anisotropic instance on `SM_Fish_A`: `sphere radius=1 subdivisions=2 scale=(4.4, 1.3, 2.5)` measures half-extents `(2.54, 0.75, 1.44)` against a nominal `(4.4, 1.3, 2.5)`, every axis short by `1/sqrt(3) = 0.5774`. Consequence: a caudal fin root placed against the nominal radius sat **0.5664997 uu outside** the peduncle — a detached tail on a compile that was otherwise green (`success: true`, no error, no warning naming the size). `floatingGeometry` (`floatingCount: 2`, `nearestDistance: 0.5664997`) was the sole signal. Mechanism verified in source at HEAD, not inferred: `AppendSphereBox` sets `EdgeVertices = FIndex3i(Steps...)` (`MeshPrimitiveFunctions.cpp:381`), so `subdivisions=2` is 2 vertices per cube edge — one quad per face, only the 8 cube corners exist — and `FBoxSphereGenerator::Generate` (`Generators/BoxSphereGenerator.h:39-70`) projects a corner to `sqrt(1/3)` per axis at `:53` and scales by `Radius` at `:69`, giving `r/sqrt(3)` on every axis; both projection methods agree at a corner. Plugin side: `GeometryOps_Primitives.cpp:215-243` (`GenerateSphere`) already runs a `ClampRangeWarn` at `:227` whose entire purpose is to report this floor, and warns nothing about the extent it produces; the `subdivisions` doc text (`PwModelParser.cpp:468`) documents the triangle count and the floor but not the factor. Fix is either a warning at `:227` or a clause on `:468`. Worked around by treating the half-extent as `radius/sqrt(3)` and moving the fin roots inside the measured box, with the factor written into `SM_Fish_A.pwmodel`'s header; defect untouched. Deduped: NO-MATCH board-wide (1316 tickets) — `floatingGeometry`/`floatingCount`/`nearestDistance` have zero occurrences anywhere, and the one adjacent ticket, `B-blossom-pole-ring-degenerate-triangles`, cites `sphere` = `AppendSphereBox` only to argue its own defect class cannot recur here; it never mentions extent, `radius=`, `subdivisions=` or `bounds`. This ticket narrows that reassurance rather than contradicting it.
