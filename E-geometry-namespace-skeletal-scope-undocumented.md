---
id: E-geometry-namespace-skeletal-scope-undocumented
title: "geometry.* advertises ~90 verbs with zero skeletal coverage and never says so — the StaticMesh/DynamicMeshActor scope boundary is invisible from outside"
status: OPEN
severity: Low
category: ergonomic
tags: [geometry, geometry-script, skeletal-mesh, docs, discoverability, scope-boundary]
---

# geometry.* is StaticMesh / DynamicMeshActor only, and the docs never say it

`Source/PinWrightGeometry/` registers roughly **90 `geometry.*` verbs** built directly on Geometry
Script. Grepping `SkeletalMesh|BoneWeight|SkeletalMeshFunctions` across the whole of
`Source/PinWrightGeometry/` returns **zero matches**. The entire typed Geometry Script surface is
StaticMesh / DynamicMeshActor; there is no typed access to `MeshBoneWeightFunctions.h` or to the
skeletal half of `MeshAssetFunctions.h`.

Line numbers and counts as read at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`
(`Plugins/PinWright/`). The registered families:

| File | Verbs |
|---|---|
| `PrimitiveHandler.cpp` | 16 — `create_box/sphere/cylinder/cone/capsule/torus/plane/disc/stairs/spiral_stairs/ring/arch/pipe/ramp/procedural_mesh`, `revolve` |
| `MeshOpsHandler.cpp` | 33 — extrude/inset/outset/bevel/shell/deformers/remesh/UV/normals/collision/`convert_to_static_mesh` |
| `BooleanHandler.cpp` | 6 |
| `MeshInfoHandler.cpp` | 12 |
| `AdvancedMeshOpsHandler.cpp` | 6 |
| `GeometryTransformHandler.cpp` | 3 |
| `MeshIOHandler.cpp` | 4 |
| `LODCollisionHandler.cpp` | 3 |
| `MeshMeasureHandler.cpp` | 2 |
| `MeshAssetIOHandler.cpp` | 1 — `create_from_static_mesh` |
| `BulkEditHandler.cpp` | 1 |

The handlers operate on `UDynamicMesh` / `ADynamicMeshActor` throughout
(`MeshAssetIOHandler.cpp:45,109,115,191,204,383`), and the one asset-ingest verb is
static-mesh-only: `:55` includes `"GeometryScript/MeshAssetFunctions.h"` and `:406` calls
`UGeometryScriptLibrary_StaticMeshFunctions::CopyMeshFromStaticMesh`.

## Why this is worth a ticket

This is not a defect — it is a scope boundary that is **invisible from outside**. An agent reading
"PinWright exposes ~90 Geometry Script verbs" reasonably concludes Geometry Script is covered, and
Geometry Script itself absolutely does skeletal meshes. The agent discovers the boundary only after
planning around it and failing.

Compounding it, this host project's *successful* skeletal pipeline is entirely undiscoverable
through the plugin: it lives in scratchpad `.py` files driven by `python.execute`, so the working
recipe is invisible to the next caller too.

Note this refutes the broader framing that "Geometry Script is reachable only through
`python.execute`" — it is extensively typed. The accurate and much sharper statement is that the
**skeletal subset** has zero typed coverage.

## Fix (docs only)

In the `docs/wiki-src/geometry.md` overlay, state the namespace scope explicitly at the top:
`geometry.*` operates on `UDynamicMesh` / `ADynamicMeshActor` and StaticMesh assets; skeletal
meshes, bone weights and bone hierarchies are **not** covered by any `geometry.*` verb. Point
readers at `skeleton.*` for editing an existing `USkeletalMesh`, and say plainly that authoring a
new one is not currently possible through any verb (cross-ref `F-geometry-skeletal-mesh-roundtrip-verbs`).
A matching line in the `skeleton.*` overlay — "`geometry.*` cannot read or write skeletal geometry"
— closes the loop from the other direction, since that is the namespace a caller is in when the
question arises.

severity rationale: impact=pure discoverability friction, fixed entirely in the wiki overlay (the
capability half is a separate ticket) × reach=`geometry.*` is a large, frequently consulted
namespace, but the specific wrong assumption is not an every-session event -> Low

## Relationship to other tickets

- `F-geometry-skeletal-mesh-roundtrip-verbs` — the capability half. Filed separately because the
  remedies are unrelated: this one is a paragraph in an overlay and can land immediately; that one
  is ~3 verbs and ~350-500 lines. Closing the feature does not remove the need to state the
  boundary, since bone-hierarchy and per-bone-weight verbs would still be absent.
- `F-static-mesh-bake-transform` (existing) already records a neighbouring shape of the same
  confusion — "`the geometry.*` namespace is a DynamicMeshActor pipeline with no StaticMesh
  ingest" — which suggests the scope statement belongs in the overlay generally, not only for the
  skeletal case.

## History
- `#1-triage-zero-skeletal-references` `OPEN` reporter — Found during a mesh/skeletal authoring triage at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`. Grepping `SkeletalMesh|BoneWeight|SkeletalMeshFunctions` across all of `Source/PinWrightGeometry/` returns zero matches, against ~90 registered `geometry.*` verbs enumerated per handler file in the body; the handlers are `UDynamicMesh`/`ADynamicMeshActor` throughout (`MeshAssetIOHandler.cpp:45,109,115,191,204,383`) and the sole asset ingest is `UGeometryScriptLibrary_StaticMeshFunctions::CopyMeshFromStaticMesh` (`:406`, include at `:55`). No verb doc or overlay states this boundary, so an agent reasonably infers skeletal coverage from the verb count and learns otherwise only after the work fails — which is what pushed this project's skeletal authoring into `python.execute`. Also records the correction that Geometry Script is *not* python-only in this plugin; it is extensively typed, just not for skeletal. Fix is docs-only: a scope paragraph in `docs/wiki-src/geometry.md` plus a reciprocal line in the `skeleton.*` overlay. Capability half tracked as `F-geometry-skeletal-mesh-roundtrip-verbs`.
