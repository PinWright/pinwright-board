---
id: B-pwmodel-bounds-orphan-vertex-after-boolean
title: "`model.validate` / `model.compile` report the UNCUT `bounds` after a boolean cuts `revolve` geometry — the box walks the vertex buffer, so a vertex the cut left referenced by no triangle still votes; the same solid written with `cylinder` reports correctly, and the number returned is exactly the extent you asked the generator for, so it reads as 'the boolean did nothing'"
status: OPEN
severity: High
category: bug
tags: [pwmodel, bounds, model.validate, model.compile, revolve, boolean, subtract, orphan-vertex, GetBounds, silent-wrong-data, reporting-fault]
encounters: 1
lastSeen: 2026-08-27T18:57:03+05:00
---

# The reported box is the box before the cut, and only on `revolve`

`model.validate` / `model.compile` report a `bounds` box that still spans the **uncut** extent after
a `subtract` has removed the end of a solid built by `revolve`. The identical solid written with
`cylinder` instead reports the correct, cut box. `signedVolume` is identical in both, so the two
meshes really are the same solid — **only the reported box differs**.

The failure mode is the dangerous one: the value returned is *exactly the extent the author asked the
generator for*, so it does not read as a bad measurement. It reads as **"the boolean did nothing"**.

## Verbatim repro

Two documents, same solid (an 8-gon prism r=100, z 0..200), same tool (a box whose underside sits at
z=150), one `model.validate` each. Measured 2026-08-27, UE 5.8, this checkout:

```
part a {                                                    ->  bounds.max.z = 200   WRONG
    revolve steps=8 profile=[(0,0),(100,0),(100,200),(0,200)]     30 triangles
    subtract { box size=(400,400,200) at=(0,0,250) }              signedVolume 4242640.687
}

part a {                                                    ->  bounds.max.z = 150   correct
    cylinder radius=100 height=200 segments=8 at=(0,0,100)        46 triangles
    subtract { box size=(400,400,200) at=(0,0,250) }              signedVolume 4242640.687
}
```

**The proof that the meshes agree and only the report differs:** `4,242,640.687` is exactly the
analytic volume of that 8-gon at height **150** — `0.5 * 8 * 100^2 * sin(45deg) * 150 = 4242640.687`
— and both documents return it to the last digit. Both solids are cut at 150. Only the `revolve` one
misreports its box.

Supporting readings on the cut `revolve` mesh: `isClosed: true`, `boundaryEdges: 0`,
`degenerateTriangles: 0`, `nonManifoldVertices: 0`, 30 triangles. A sound closed prism of height 150
that claims to be 200 tall.

**No in-format remedy exists.** `weld_vertices` after the `subtract` does **not** clear it: bounds
still 200, still 30 triangles.

## The decisive control: the baked asset is fine

`static_mesh.describe` on `/Game/Atlantis/Meshes/SM_Column_Broken_A`, whose source `bounds` claimed
**z max 1800**, returns `origin.z 739.38` / `extent.z 739.56` — a real span of **z -0.18 .. 1478.9**.
The bake drops whatever the model verbs are counting.

So this is purely a **reporting** fault on `model.validate` / `model.compile`, not a geometry fault.
Nothing wrong is written to disk; the number the author steers by is wrong.

## Root cause

**Source-verified half** — the bounds reduction walks the **vertex buffer**, with no reference to
triangles, at both emission sites:

`Plugins/PinWright/Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:3303` (model-wide) and
`:2954` (per-part):

```cpp
    const UE::Geometry::FAxisAlignedBox3d MergedBox = Merged->GetMeshRef().GetBounds();
```

and the engine's `FDynamicMesh3::GetBounds`
(`C:/UE_5.8/Engine/Source/Runtime/GeometryCore/Private/DynamicMesh/DynamicMesh3_Queries.cpp:604-625`)
is documented *"Computes bounding box of all vertices"* and iterates `VertexIndicesItr()` /
`IsVertex(vid)`:

```cpp
	for (int vi : VertexIndicesItr())
	{
		MinVec = Min(MinVec, Vertices[vi]);
		MaxVec = Max(MaxVec, Vertices[vi]);
	}
```

There is no triangle in that loop. **Any vertex still allocated in the buffer votes, whether or not a
triangle references it.** That is the mechanism by which a stale vertex can inflate the reported box,
and it is fact, not hypothesis. Emission of the field is `Handlers/Model/ModelCompileHandler.cpp:683-685`
(model) and `:824-826` (per-part).

**Unverified hypothesis — the remaining step.** *Which* vertex survives at z=200 was **not traced
into plugin or engine source.** The inference from the readings is that `revolve` closes its ends
with an axis fan sharing one apex vertex (the repro profile has points **on** the axis at `(0,0)` and
`(0,200)`), the `subtract` removes every triangle that referenced the upper apex but leaves the
vertex allocated, and `GetBounds` then counts it. That is consistent with all four readings — closed,
0 boundary edges, 30 triangles, `weld_vertices` unable to clear it (an isolated vertex has no edge to
weld along) — and with `cylinder` being unaffected, since `AppendCylinder`'s caps do not produce an
on-axis apex from an author-supplied profile. **It is inference from the response surface, not a
source read; do not treat the apex-fan detail as established.** `revolve` reaches
`AppendRevolvePath` via `GeometryOps_Primitives.cpp:815-857` if a fixer wants to confirm it.

## Impact

It defeats the workflow the docs prescribe. `model.authoring` § *Validate, compile, inspect* says
size is "the one check that does not need step 5" and tells the author to "fit a model to a required
bounding box there, on the surface that creates nothing, rather than writing an asset per iteration".

For any model whose silhouette is **cut out of a `revolve`** — which is every broken column, every
collapsed dome and every snapped shaft on this build — that surface returns the wrong number, and it
fails **silently and plausibly**, because the wrong number is exactly the one the generator was
asked for. It cost four diagnostic compiles plus a render on `SM_Column_Broken_A`, chasing a break
that had in fact worked the whole time.

It also mis-sizes any collision element an author dimensions from the reported box.

## Workaround

Do not trust `bounds` on a part whose `revolve` was cut by a boolean. Either:

- cross-check `signedVolume` against the analytic volume of the intended solid, or
- compile and read `static_mesh.describe`'s `origin` / `extent`, which are correct.

Both are worse than the check they replace: one needs arithmetic outside the format, the other needs
exactly the asset write that `bounds` exists to avoid.

## Suggested fix

Compute the reported bounds from **referenced** vertices only, or compact the vertex buffer after a
boolean before measuring. Either makes the `revolve` and `cylinder` spellings agree, which is the
invariant that is broken here. The engine offers `GetBoundsForTriangleSelection`
(`DynamicMesh3.h:1197`) if a triangle-driven reduction is wanted without a compaction pass.

## Distinct from related tickets

- **The `.pwmodel` `bounds` response field appears in ZERO board tickets.** A read-only sweep of all
  1316 confirmed it. This ticket opens that category; nothing to dedup against.
- Three `bounds`-adjacent tickets were checked and ruled out — all are in the `geometry.*` **RPC**
  namespace, a different surface with a different fault:
  `E-geometry-mesh-info-omits-bbox` (IN-REVIEW, Low) is `get_mesh_info` **omitting** a box that this
  surface emits; `B-mesh-info-empty-nonfinite-json` (IN-REVIEW, Medium) is a non-finite box on an
  **empty** mesh serializing as invalid JSON; `B-revolve-polygon-drops-material-id` (OPEN, Medium)
  shares the word `revolve` and nothing else — it is `MaterialID` on `append_revolve_polygon`.
- `B-pwmodel-sphere-subdivisions-extent` (filed in this same run) is the other extent-versus-parameter
  trap on this surface, and is its mirror image: there `bounds` is **correct** and the author's
  reading of `radius=` is wrong; here `bounds` is **wrong**. Fixing one says nothing about the other.

severity rationale: impact=silent wrong data on a normal path — `bounds` returns the pre-cut extent and the caller builds on it; the caller trusts a result that is a lie, and it lies in the single most persuasive way available (returning exactly the value the author asked the generator for, so it reads as a no-op boolean rather than a bad measurement) × reach=`model.validate` is the surface the docs prescribe for iterating a model to a size, and the affected construction (a silhouette cut out of a `revolve`) is every broken column, collapsed dome and snapped shaft on this build -> High. Not Critical: nothing crashes and nothing incorrect reaches disk — `static_mesh.describe` on the baked asset is correct.

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Two-document repro on the same solid (8-gon prism r=100, z 0..200; tool = box underside at z=150): `revolve steps=8 profile=[(0,0),(100,0),(100,200),(0,200)]` + `subtract` reports **`bounds.max.z = 200`** (30 triangles); `cylinder radius=100 height=200 segments=8 at=(0,0,100)` + the same `subtract` reports **`bounds.max.z = 150`** (46 triangles). `signedVolume` is **4,242,640.687 in BOTH**, and that is exactly the analytic volume of that 8-gon at height 150 (`0.5*8*100^2*sin(45deg)*150`) — so both meshes are the same solid, cut at 150, and only the reported box differs. Cut `revolve` mesh reads `isClosed: true`, `boundaryEdges: 0`, `degenerateTriangles: 0`, `nonManifoldVertices: 0`. `weld_vertices` after the subtract does NOT clear it (bounds still 200, still 30 triangles), so there is no in-format remedy. **Decisive control — the baked asset is fine:** `static_mesh.describe` on `/Game/Atlantis/Meshes/SM_Column_Broken_A`, whose source `bounds` claimed `z max 1800`, returns `origin.z 739.38` / `extent.z 739.56` (real span z -0.18..1478.9), so this is purely a REPORTING fault on the two model verbs, not a geometry fault. Root cause, source-verified half: bounds is `GetMeshRef().GetBounds()` at `PwModelCompiler.cpp:3303` (model-wide) and `:2954` (per-part), emitted at `ModelCompileHandler.cpp:683-685` / `:824-826`; and `FDynamicMesh3::GetBounds` (`DynamicMesh3_Queries.cpp:604-625`, UE 5.8) is documented "Computes bounding box of all vertices" and loops `VertexIndicesItr()` with no reference to triangles — so any vertex still allocated votes whether or not a triangle uses it. **Root cause NOT fully traced: which vertex survives at z=200 was not read in source.** The hypothesis — `revolve` closes its ends with an axis fan sharing one apex (the repro profile has on-axis points at `(0,0)` and `(0,200)`), the `subtract` orphans the upper apex, and the vertex-walking reduction still counts it — is inference from the readings, consistent with all four of them and with `cylinder` being unaffected, and is explicitly marked unverified in the body. `revolve` reaches `AppendRevolvePath` via `GeometryOps_Primitives.cpp:815-857` for a fixer who wants to confirm it. Impact: defeats the workflow `model.authoring` § "Validate, compile, inspect" prescribes — fit a model to a bounding box on the surface that creates nothing rather than writing an asset per iteration — for every silhouette cut out of a `revolve`, and fails plausibly because the wrong value is exactly the extent asked of the generator, reading as "the boolean did nothing". Cost four diagnostic compiles plus a render on `SM_Column_Broken_A`. Suggested fix: measure referenced vertices only, or compact the vertex buffer after a boolean before measuring (`GetBoundsForTriangleSelection`, `DynamicMesh3.h:1197`, if a triangle-driven reduction is preferred). Deduped: NO-MATCH board-wide (1316 tickets) — the `.pwmodel` `bounds` response field appears in ZERO tickets. Ruled out: `E-geometry-mesh-info-omits-bbox`, `B-mesh-info-empty-nonfinite-json`, `B-revolve-polygon-drops-material-id` — all `geometry.*` RPC namespace, different faults. Worked around by cross-checking `signedVolume` and `static_mesh.describe`; defect untouched.
