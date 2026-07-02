---
id: B-set-vertex-color-wrong-channel-not-persisted
title: "geometry.set_vertex_color writes to the legacy FDynamicMesh3 per-vertex color buffer, not the attribute-overlay PrimaryColors channel — returns verticesModified success but get_mesh_info still reports hasColors:false and the tint is dropped by rendering and the StaticMesh bake"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, set_vertex_color, vertex-color, dynamic-mesh, legacy-vs-overlay, primary-colors, silent-noop, success-no-effect]
encounters: 1
lastSeen: 2026-07-02T07:54:26.7859136+03:00
---

# `geometry.set_vertex_color` writes colors to the wrong storage channel — success is reported but the color never lands where the rest of the pipeline reads it

`geometry.set_vertex_color` (`MeshInfoHandler.cpp:414-484`) enables and writes
vertex colors through the **legacy per-vertex color buffer** on `FDynamicMesh3`:

```cpp
UE::Geometry::FDynamicMesh3& EditMesh = Target.Mesh->GetMeshRef();
// Enable vertex colors if not already enabled
if (!EditMesh.HasVertexColors())
{
    EditMesh.EnableVertexColors(FVector3f(1.0f, 1.0f, 1.0f));   // line 448 — legacy buffer
}
...
EditMesh.SetVertexColor(VID, Color);                            // line 458 — legacy buffer (setAll)
...
EditMesh.SetVertexColor(VertexIndex, Color);                   // line 464 — legacy buffer (single)
```

`FDynamicMesh3::EnableVertexColors` / `HasVertexColors` / `SetVertexColor` operate
on the mesh's **legacy dense per-vertex color array**, which is a *different*
storage location from the mesh **attribute-overlay** color channel
(`FDynamicMesh3::Attributes()->PrimaryColors()`). The rest of the GeometryScript
ecosystem — the DynamicMeshComponent renderer, the `convert_to_static_mesh` bake,
and this plugin's own `get_mesh_info` reader — consume the **attribute overlay**,
not the legacy buffer.

The handler's own sibling reader proves the mismatch. `geometry.get_mesh_info`
(`MeshInfoHandler.cpp:98`) reports `hasColors` via:

```cpp
bool bHasVertexColors = UGeometryScriptLibrary_MeshQueryFunctions::GetHasVertexColors(Target.Mesh);
```

and the engine implements that as an attribute-overlay check (verbatim,
`Engine/Plugins/Runtime/GeometryScripting/Source/GeometryScriptingCore/Private/MeshQueryFunctions.cpp:742-744`):

```cpp
return SimpleMeshQuery<bool>(TargetMesh, false, [&](const FDynamicMesh3& Mesh) {
    return (Mesh.HasAttributes() && Mesh.Attributes()->PrimaryColors() != nullptr);
});
```

So `set_vertex_color` fills the legacy buffer, `EnableVertexColors` never touches
`Attributes()->PrimaryColors()`, and `GetHasVertexColors` correctly returns
`false` — the tint is invisible to the query, to the viewport (the component
renders overlay colors), and to `convert_to_static_mesh` (GeometryScript's
mesh→StaticMesh copy reads the color overlay). The caller is handed a clean
success (`verticesModified:128`, `"Vertex color set"`) for an operation that
produced no consumable effect: a **silent false-success**.

This is the `success-no-effect` tool-bug class on a valid, documented input, but
distinct in root cause from the other members of that family (this one is a
wrong-storage-channel write in `FDynamicMesh3`, not a dropped param or a no-op
handler). No existing board ticket covers `set_vertex_color` or the
legacy-buffer-vs-attribute-overlay color dichotomy.

## Repro (verbatim, replay-confirmed against mcp__pinwright__call)

```
geometry.create_torus {name:"ReplayRing", majorRadius:200, minorRadius:30, majorSegments:16, minorSegments:8}
  -> {name:"ReplayRing", class:"DynamicMeshActor"}

geometry.get_mesh_info {actorName:"ReplayRing"}
  -> {vertexCount:128, triangleCount:256, hasColors:false, ...}      baseline, no colors

geometry.set_vertex_color {actorName:"ReplayRing", setAll:true, r:0.82, g:0.45, b:0.18, a:1}
  -> {verticesModified:128, r:0.82, g:0.45, b:0.18, a:1, message:"Vertex color set"}   SUCCESS

geometry.get_mesh_info {actorName:"ReplayRing"}
  -> {vertexCount:128, triangleCount:256, hasColors:false, ...}      STILL false — tint went to the legacy buffer, not the overlay
```

Same actor, same `Target.Mesh` handle in both calls, so this is not a cross-actor
mixup — the color write and the color read address the same mesh and still
disagree.

## What it should do

`set_vertex_color` should write to the **attribute-overlay** color channel that
the query, renderer, and bake all read. Options:

1. Use GeometryScript's `UGeometryScriptLibrary_MeshVertexColorFunctions::
   SetMeshPerVertexColors` (writes `Attributes()->PrimaryColors()`), or
2. Before writing, enable attributes + the primary-color overlay
   (`EditMesh.EnableAttributes()` if needed; `EditMesh.Attributes()->
   EnablePrimaryColors()`) and set overlay elements per vertex, instead of the
   legacy `EnableVertexColors`/`SetVertexColor` path.

After the fix, `get_mesh_info` should report `hasColors:true` and the tint should
survive `convert_to_static_mesh` into the baked asset.

**Workaround (pre-fix):** none from the RPC surface — there is no exposed way to
write the attribute-overlay color channel, so a vertex tint applied via
`set_vertex_color` cannot be made to render or bake, and `get_mesh_info` gives
the caller the correct (but seemingly contradictory) `hasColors:false` readback.

severity rationale: impact=silent-false-success (the caller is told 128 vertices
were colored on a normal, documented geometry verb, but the tint is dropped by
the query, the viewport, and the bake) × reach=normal documented path (not rare)
-> High

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against
  mcp__pinwright__call on a fresh `create_torus` (majorR=200/minorR=30, 128v/256t).
  `set_vertex_color {setAll:true, r:0.82,g:0.45,b:0.18}` returns
  `verticesModified:128` / `"Vertex color set"` but `get_mesh_info` reports
  `hasColors:false` both before and after (same actor, same `Target.Mesh`). Root
  cause: the handler writes the **legacy** `FDynamicMesh3` per-vertex color buffer
  via `EnableVertexColors`/`SetVertexColor` (`MeshInfoHandler.cpp:448,458,464`),
  while `get_mesh_info`'s `GetHasVertexColors` (`MeshInfoHandler.cpp:98`) checks the
  **attribute-overlay** channel `Attributes()->PrimaryColors()` (engine
  `MeshQueryFunctions.cpp:742-744`) — two different storage locations. The
  attribute overlay is also what the DynamicMeshComponent renders and what
  `convert_to_static_mesh` bakes, so the tint is silently dropped everywhere the
  color would matter. Surfaced by a "forge a warm-tinted portal-ring prop and bake
  it to a StaticMesh" task whose attempt agent noticed `set_vertex_color` reported
  `verticesModified` success yet `hasColors` stayed false on re-apply, and flagged
  it as a possible persistence mismatch (it was not part of the task's dimensional
  success-check, so it did not block completion).
- `#2-write-attribute-overlay` `IN-REVIEW` developer — Retargeted
  `geometry.set_vertex_color` from the legacy `FDynamicMesh3` per-vertex color buffer
  to the attribute-overlay `PrimaryColors()` channel that `get_mesh_info`
  (`GetHasVertexColors`), the DynamicMeshComponent renderer, and the
  `convert_to_static_mesh` bake all read. In
  `Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/MeshInfoHandler.cpp`
  the handler now `EnableAttributes()` + `Attributes()->EnablePrimaryColors()`, seeds a
  fresh overlay with one neutral-white element per vertex (triangles wired), then
  `setAll` sets every overlay element and the single-vertex path sets only the elements
  whose parent vertex matches — mirroring the engine's own
  `UGeometryScriptLibrary_MeshVertexColorFunctions::SetMeshConstantVertexColor`. The
  `verticesModified` response contract is preserved (`EditMesh.VertexCount()` for
  setAll, 1 for single). Added regression test
  `PinWright.geometry.set_vertex_color.PersistsToAttributeOverlay`
  (`Plugins/PinWright/Source/PinWright/Private/Tests/Geometry/TestGeometrySetVertexColorPersistsToOverlay.cpp`):
  spawns an in-code `create_box` DynamicMeshActor, asserts baseline `hasColors:false`,
  runs `set_vertex_color {setAll:true}`, then asserts `get_mesh_info` reports
  `hasColors:true` — which fails against the reverted legacy-buffer write (leaves
  `PrimaryColors()` null).
