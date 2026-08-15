---
id: B-revolve-polygon-drops-material-id
title: "append_revolve_polygon silently drops PrimitiveOptions.MaterialID (box/sphere -> 1, revolve -> 0)"
status: OPEN
severity: Medium
category: bug
tags: [geometry-script, upstream-engine, material-id, silent-wrong-data]
encounters: 2
lastSeen: 2026-08-15T17:21:12+05:00
---

# append_revolve_polygon silently drops PrimitiveOptions.MaterialID (box/sphere -> 1, revolve -> 0)

`GeometryScriptPrimitiveOptions.material_id` is honoured by the box/sphere primitive
family and **silently ignored** by the revolve family. Every triangle a revolve emits
lands in material slot **0** regardless of what the caller asked for, and the call
reports success.

Measured on UE 5.8, same options object passed to all three primitives, read back with
`get_max_material_id`:

| Primitive | `material_id` requested | `get_max_material_id` |
|---|---|---|
| `append_box` | 1 | `(1, True)` |
| `append_sphere_box` | 1 | `(1, True)` |
| `append_revolve_polygon` | 1 | **`(0, True)`** |

A mesh assembled from mixed primitives therefore comes out with inconsistent material
assignment and the caller gets no indication why: the baked asset still carries the
expected number of material slots and the actor still has the expected materials
assigned, so nothing looks wrong until the revolved surface renders with the wrong
material.

## The fault is in the engine, not in PinWright's wrapping

Confirmed by reading engine source; no build required. Two near-identical option-applying
helpers exist in
`Engine/Plugins/Runtime/GeometryScripting/Source/GeometryScriptingCore/Private/MeshPrimitiveFunctions.cpp`,
and only one of them applies `MaterialID`:

- `ApplyPrimitiveOptionsToMesh` (`:36-85`) applies `PreTranslate`/`PreRotate`, the
  transform, `PolygroupMode`, `bFlipOrientation`, **and** `MaterialID` (`:73-84`,
  guarded `if (PrimitiveOptions.MaterialID > 0)`).
- `AppendPrimitiveMesh` (`:158-212`) is a copy of the same option handling in a local
  lambda (`:165-192`) that **omits the MaterialID block entirely**. No other code path
  reinstates it.

`AppendBox` / `AppendSphereBox` reach the mesh through `AppendPrimitive` (`:87-114`) ->
`ApplyPrimitiveOptionsToMesh`, so MaterialID is applied. `AppendRevolvePolygon`
(`:731-783`) ends at `AppendPrimitiveMesh` (`:781`), so it is dropped. That is exactly
the 1 / 1 / 0 split measured above.

Five entry points end at `AppendPrimitiveMesh` and are affected identically:

- `AppendRevolvePolygon` (`:781`)
- `AppendSpiralRevolvePolygon` (`:838`)
- `AppendRevolvePath` (`:896`)
- `AppendTriangulatedPolygon3D` (`:1381`)
- `AppendSimpleCollisionShapes` (`:1849`)

plus `AppendTorus` (`:708`) by delegation — it tail-calls `AppendRevolvePolygon` at
`:727`.

Long-standing, not a 5.8 regression: `AppendPrimitiveMesh` contains zero references to
`MaterialID` in **UE 5.4, 5.5, 5.6, 5.7 and 5.8** (checked against each installed engine
tree). This is an upstream report, not a PinWright code defect.

## PinWright's exposure is latent, not live

No PinWright verb currently trips this, because no primitive verb exposes a material ID
at all — every `FGeometryScriptPrimitiveOptions` in `PrimitiveHandler.cpp` is
default-constructed (`MaterialID` 0), and the only `materialId` parameter in the geometry
namespace belongs to `geometry.append_buffers` (`BulkEditHandler.cpp:188`), which is a
different code path. The verbs sitting on the broken helper are:

- `geometry.revolve` -> `AppendRevolvePath`
- `geometry.create_torus`, `geometry.create_arch` -> `AppendTorus` -> `AppendRevolvePolygon`

They would start dropping the ID the day a `materialId` parameter is added to them
(**F-geometry-mesh-material-assign** touches this area). The live exposure today is
`python.execute` callers driving Geometry Script directly, which is the documented
mesh-authoring route for this project.

Origin: found during a map-authoring session, `Docs/map/modern_landmarks.md` (section C,
"Engine bug found and worked around", and section on the Twin Gates kit). **Independently
reproduced twice** — once building the Tormentor dais/seams and again, without reference
to the first, building the Twin Gates iris and trough water surface. Stable, not
environment-specific.

severity rationale: impact=silent wrong data on a normal path (the caller trusts an option that was discarded and ships an asset with the wrong material slot) -> High; reach=no PinWright verb exposes materialId today, so it only fires for `python.execute` Geometry Script callers who pass a non-zero id to a revolve, and it has a short reliable workaround -> bump down -> Medium.

**Workaround:** Do not rely on `material_id` for revolved geometry. Accumulate each
non-zero material ID into its **own** `DynamicMesh`, then call `enable_material_i_ds` +
`remap_material_i_ds(0 -> id)` on it before appending to the combined mesh. That is
immune to which primitive honours the option. Shipped as
`Docs/scripts/.../tm_assets.py` `Build`/`finish()`; `scratchpad/dota_arch.py` still uses
the naive `opts(mat)` form and its revolve-only accents are consequently untagged.

**Fix:** Two parts, independent.

1. *Upstream (the real fix).* Report to Epic: `AppendPrimitiveMesh`
   (`MeshPrimitiveFunctions.cpp:158-212`) should apply `PrimitiveOptions.MaterialID` the
   way `ApplyPrimitiveOptionsToMesh` (`:73-84`) does — ideally by deleting the duplicated
   lambda and calling the shared helper, since the divergence is a copy-paste omission
   and will drift again otherwise.
2. *Here.* Normalise or document, per verb. If a `materialId` parameter is ever added to
   `geometry.revolve` / `create_torus` / `create_arch`, do **not** pass it through
   `FGeometryScriptPrimitiveOptions` — apply it after the append via the
   `enable_material_i_ds` + `remap_material_i_ds` route so the verb's contract holds
   regardless of engine version. Until then, add the difference to the
   `geometry.revolve` / `geometry.create_torus` / `geometry.create_arch` method pages
   (`docs/wiki-src/geometry.md`): Geometry Script's primitive `MaterialID` option is
   dropped by the revolve family, so material IDs on revolved geometry must be assigned
   after the fact.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a map-authoring session (`Docs/map/modern_landmarks.md`). Repro on UE 5.8 via `python.execute`: build one `DynamicMesh`, pass a single `GeometryScriptPrimitiveOptions` with `material_id = 1` to `append_box`, `append_sphere_box` and `append_revolve_polygon` in turn, and read `get_max_material_id` after each — box `(1, True)`, sphere `(1, True)`, revolve `(0, True)`. Root cause confirmed in engine source without a build: `AppendPrimitiveMesh` (`MeshPrimitiveFunctions.cpp:158-212`) is a copy of `ApplyPrimitiveOptionsToMesh` (`:36-85`) with the MaterialID block at `:73-84` omitted; box/sphere route through `AppendPrimitive` -> `ApplyPrimitiveOptionsToMesh` and get it, `AppendRevolvePolygon` ends at `AppendPrimitiveMesh` (`:781`) and does not. Affects `AppendRevolvePolygon`, `AppendSpiralRevolvePolygon`, `AppendRevolvePath`, `AppendTriangulatedPolygon3D`, `AppendSimpleCollisionShapes`, and `AppendTorus` by delegation; present in UE 5.4-5.8. Independently reproduced twice (Tormentor build, then Twin Gates build). Fault is upstream in the engine's Geometry Script, not in PinWright's wrapping; PinWright's exposure is latent because no primitive verb exposes a material ID yet.
