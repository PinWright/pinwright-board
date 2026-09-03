---
id: B-fill-holes-uvless-assert
title: "geometry.fill_holes and bridge's one-loop fallback crash on an attribute-enabled mesh with zero UV layers when the engine projects UVs into a missing overlay"
status: OPEN
severity: Critical
category: bug
tags: [geometry, fill-holes, bridge, uvless, assertion, editor-crash, validate-before-call]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Hole filling reaches a fatal engine UV-layer assertion on ordinary hand-authored meshes

> **SOURCE-ONLY.** The fatal chain is explicit in current plugin and UE 5.8 source; this scan did
> not invoke it in the shared editor.

`GeometryOps_Modeling.cpp:2800-2817` sends `geometry.fill_holes` directly to
`UGeometryScriptLibrary_MeshRepairFunctions::FillAllMeshHoles` without checking the primary UV
layer. `GeometryOps_Advanced.cpp:243-252` calls the same helper with `MinimalFill` when
`geometry.bridge` sees fewer than two boundary loops.

The engine copies the mesh and, after a successful non-triangle hole fill, tests only
`ResultMesh->HasAttributes()` before calling `FDynamicMeshEditor::SetTriangleUVsFromProjection`
(`HoleFillOp.cpp:438-445`). That function hard-checks
`Mesh->HasAttributes() && NumUVLayers() > UVLayerIndex`
(`DynamicMeshEditor.cpp:1502-1508`). An attribute-enabled mesh with zero UV layers passes the first
gate and fails the second, aborting the editor.

This state is a normal PinWright product: `append_buffers` without `uvs` sets the target UV-layer
count to zero while retaining attributes. The same file already carries guards for the identical
shape in bevel and shell, and even tells a UV-less shell caller to "close the mesh with
fill_holes" (`GeometryOps_Modeling.cpp:1588-1626`, `:1756-1849`), directing the caller into this
unguarded fatal path.

## What should happen

Before either `FillAllMeshHoles` call, ensure UV channel 0 exists with the shared
`GeometryUtils::EnsureMeshHasUVChannel`, or refuse with `NO_UV_ELEMENTS` before mutation. Preserve
all existing UV layers. Regression coverage must construct an attribute-enabled mesh with zero UV
layers and exercise both `fill_holes` and bridge's `<2 loops` fallback; it must fail before reaching
the engine helper if the layer cannot be created.

**Workaround:** create UV channel 0 and project UVs before `fill_holes`; do not use `bridge` as a
hole-filling fallback on a UV-less mesh.

## Related

`B-convert-static-mesh-uvless-crash` covers a different MikkT build assertion. The reusable
UV-channel guard already exists from `B-geometry-uv-gen-silent-noop`.

## History
- `#1-source-pattern-scan` `OPEN` reporter — Both PinWright call sites omit the UV-layer precondition; UE 5.8's hole filler gates only on attributes and its projection helper then asserts that UV layer 0 exists. The reachable zero-layer state is produced by `append_buffers` without `uvs`. Source-only; no editor call was made.
