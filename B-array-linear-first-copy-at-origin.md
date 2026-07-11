---
id: B-array-linear-first-copy-at-origin
title: "geometry.array_linear places the FIRST appended copy at the origin (identity transform), so it doubles the original in place and the row ends up one full spacing short of the requested length"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, array_linear, dynamic-mesh, off-by-one, array-first-copy-at-origin, array-placement]
encounters: 1
lastSeen: 2026-07-11T03:56:34.4388061+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T06:06:20.9381634+03:00
---

# `geometry.array_linear` doubles the original at the origin and drops one spacing — the first appended copy is placed with an identity transform, not offset

`geometry.array_linear` is documented (wiki + handler) as producing `count`
total copies including the original, evenly spaced by `offset`. For a linear
array of `count` copies with spacing `S`, the copies should land at
`0, S, 2S, ..., (count-1)*S` — the original at 0 and the `count-1` appended
copies at `S ... (count-1)*S`.

Instead, the FIRST appended copy is placed at the **origin** (identity
transform), coincident with the original, and only the remaining copies step
by `offset`. So the copies land at `0, 0, S, 2S, ..., (count-2)*S`: a doubled
coincident mesh at the origin plus a run that stops one spacing short. The
geometry (vertex/triangle/volume counts) still multiplies by `count` correctly,
and the success response echoes a plausible `count` — nothing signals that the
spatial layout is wrong. The caller trusts an evenly-spaced array and silently
gets a doubled origin copy and a too-short row.

## Root cause (guilty source line)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/GeometryTransformHandler.cpp:211-212`:

```cpp
    UGeometryScriptLibrary_MeshBasicEditFunctions::AppendMeshRepeated(
        Mesh, SourceMesh, RepeatTransform, Count - 1, false, false, AppendOptions, nullptr);
```

The 5th positional argument (`false`) is `bApplyTransformToFirstInstance`.
With it `false`, `AppendMeshRepeated` places instance `k` (for `k = 0 .. RepeatCount-1`)
at `k * RepeatTransform`, so the first appended instance (`k=0`) gets the
identity transform and lands on top of the original. To get proper spacing the
first appended copy must be offset by `1 * offset`, i.e. this argument must be
**`true`** — then instance `k` lands at `(k+1) * offset`, giving the full
`0, S, ..., (count-1)*S` layout. The sibling `geometry.array_radial` does NOT
share this path: it builds explicit transforms `for (i = 1; i < Count; ++i)`
(GeometryTransformHandler.cpp:296) and calls `AppendMeshTransformed`, so radial
spacing is correct. This bug is confined to `array_linear`.

## Verbatim repro (replayed live via `mcp__pinwright__call`)

Definitive minimal case — `count=2`, `offset.x=100`, a cylinder of radius 5
(local X extent 10):

```
geometry.create_cylinder {"name":"ReplayBaluster2","radius":5,"height":200,"segments":12,"location":{"x":0,"y":0,"z":100}}
geometry.array_linear   {"actorName":"ReplayBaluster2","count":2,"offset":{"x":100,"y":0,"z":0}}
  -> {"actorName":"ReplayBaluster2","count":2,"vertexCount":76,"triangleCount":144,"message":"Linear array applied"}
geometry.measure {"actorName":"ReplayBaluster2","space":"local"}
  -> bbox.size = {x:10, y:10, z:200}, volume:30000, triangleCount:144
```

`volume:30000` = 2 x 15000 and `triangleCount:144` = 2 x 72 prove two full
copies exist, but `bbox.size.x` is **10** — identical to a single baluster.
Both copies are stacked at x=0; the `offset.x=100` was never applied. Correct
result would be `bbox.size.x = 110` (min -5, max 105).

The original 8-upright task case — `count=8`, `offset.x=100`:

```
geometry.array_linear {"actorName":"ReplayBaluster","count":8,"offset":{"x":100,"y":0,"z":0}}
geometry.measure {"actorName":"ReplayBaluster","space":"local"}
  -> bbox: min.x=-5, max.x=605, size.x=610, center.x=300, volume:120000, triangleCount:576
```

`volume:120000` = 8 x 15000 (eight full copies) but `max.x=605` means the
rightmost copy is centered at 600 — copies sit at `{0 (doubled), 100, 200, 300,
400, 500, 600}`, so the run spans **610** (~6.1 m). The expected layout
`{0,100,...,700}` would give `max.x=705`, `size.x=710` (~7 m). The array is one
full 100-unit spacing short and carries a wasted coincident copy at the origin.

## What it should do

Pass `bApplyTransformToFirstInstance = true` so the first appended copy is
offset by one spacing. Then `count` copies land evenly at `0 .. (count-1)*offset`
with no doubled origin mesh, matching the documented "count total copies
including original, evenly spaced by offset" contract and the correct behavior
of `array_radial`.

## Distinct from related tickets

- `E-geometry-array-radial-merges-in-place` (IN-REVIEW, ergonomic) is about the
  wiki/name implying `array_radial`/`array_linear` spawn N separate ACTORS vs.
  merging in place; it explicitly calls the in-place merge "correct work". That
  is a docs/semantics issue — it does NOT cover this spatial off-by-one placement
  defect (doubled origin copy + one-spacing-short span), which is a genuine
  correctness bug in the merged geometry itself.
- `E-geometry-deformer-echo-mesh-counts` (missing count echo) and
  `E-duplicate-along-spline-undiscoverable` (discoverability) are unrelated.

severity rationale: impact=silent-wrong-geometry (doubled coincident copy at origin + row one spacing short, success response echoes correct-looking counts) x reach=common modeling verb -> High

## History
- `#1-initial-repro` `OPEN` reporter — Seed task `geometry.array_linear` (Victorian cemetery railing: single baluster arrayed into an 8-upright row ~1 m apart, ~7 m total, merged mesh). Replay-confirmed live via `mcp__pinwright__call`. Minimal `count=2`/`offset.x=100` case: `array_linear` on a radius-5 cylinder returned `count:2`/`triangleCount:144`, and `geometry.measure` reported `bbox.size.x=10` with `volume:30000` — two full copies stacked at x=0 (offset never applied); correct would be `size.x=110`. Task case `count=8`/`offset.x=100`: `measure` -> `max.x=605`, `size.x=610`, `volume:120000` (8 copies), i.e. copies at `{0 doubled,100,200,300,400,500,600}`, run ~6.1 m not the requested ~7 m. Root cause: `AppendMeshRepeated(..., Count-1, false /*bApplyTransformToFirstInstance*/, ...)` at GeometryTransformHandler.cpp:211-212 places the first appended instance at identity; must be `true` so copies land at `0 .. (count-1)*offset`. Confined to `array_linear` (sibling `array_radial` uses explicit `i=1..Count-1` transforms via `AppendMeshTransformed`, spacing correct). Classified TOOL BUG (silent wrong geometry on a normal path).
- `#2-triage` `IN-REVIEW` developer — GO. Validity confirmed against engine source: `GeometryTransformHandler.cpp:212` passes `AppendMeshRepeated`'s 5th positional arg `bApplyTransformToFirstInstance=false` (engine default is `true`, `MeshBasicEditFunctions.h:396`; impl `MeshBasicEditFunctions.cpp:748-765`), so the first appended copy takes the identity transform and doubles the original at the origin while the row ends one full spacing short. Decision: flip that arg to `true` — the minimal root-cause fix. Not a duplicate (the two IN-REVIEW `E-` siblings are docs/count-echo, already in HEAD at :152/:222, orthogonal lines), not a regression (`git log -S` shows the arg was original, never a DONE re-break). `array_radial` unaffected; `Count=1`→`RepeatCount=0` no-op preserved. Shipped: flipped the 5th positional arg to `/*bApplyTransformToFirstInstance=*/ true` in `Source/PinWright/Private/Handlers/Geometry/GeometryTransformHandler.cpp` (array_linear handler, line 212). Plugin compiled clean (Result: Succeeded). Adopted red test `PinWright.geometry.array_linear.FirstCopyOffsetBySpacing` (`Source/PinWright/Private/Tests/Geometry/TestGeometryArrayLinearFirstCopyOffset.cpp`) flipped Fail→Success post-fix — red-green differential holds by construction.
