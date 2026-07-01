---
id: B-mesh-info-empty-nonfinite-json
title: "geometry.get_mesh_info on an empty (0-vertex) mesh serializes a non-finite boundingBox extent as invalid JSON — the client rejects the whole response and reports it as 'Editor not reachable'"
status: OPEN
severity: Medium
category: bug
tags: [geometry, get_mesh_info, bounding-box, empty-mesh, non-finite, nan, inf, malformed-json, serialization, editor-down]
encounters: 1
lastSeen: 2026-07-01T08:47:34.8405347+03:00
---

# get_mesh_info on an empty mesh emits invalid JSON (non-finite bounding-box extent), which the MCP client mis-reports as editor-down

`geometry.get_mesh_info {actorName}` unconditionally serializes the mesh's
local-space bounding box (`min`/`max`/`origin`/`extent`) into its response. For
an **empty** DynamicMesh (0 vertices — e.g. a fresh `geometry.create_procedural_mesh`
actor, or a mesh emptied by a boolean/delete/simplify), `GetMeshBoundingBox`
returns the inverted "empty" `FBox` (`Min` = `+DBL_MAX`, `Max` = `-DBL_MAX`).
`MeshBounds.GetExtent()` = `(Max - Min) * 0.5` then **overflows to a non-finite
value** — `Max - Min` = `-3.6e308` exceeds the double range, so `extent` becomes
`-inf`. `BuildVectorJson` writes that raw double via `SetNumberField` with no
finite check, and UE's JSON writer emits a bare `inf` / `nan` token. Bare
`inf`/`nan` is **not valid JSON** (JSON has no non-finite literals), so the
entire response is unparseable. The MCP client's `json.loads` throws
`Expecting value`, and the transport bridge surfaces that as **"Editor not
reachable ... launch the Unreal editor and retry"** — i.e. it looks like the
editor died, when it is up and every other call succeeds.

This is a malformed-response (non-JSON) tool bug, and the disguise is the
dangerous part: a single bad call on an empty mesh masquerades as `editor_down`,
which in an autonomous loop can trigger a needless editor kill/restart or abort a
run. `get_mesh_info` is itself an every-session verb (it is the standard readback
for the whole `geometry` namespace), so any authoring path that leaves a mesh
momentarily empty and then reads its info hits this.

Note: this bounding-box serialization was added by `E-geometry-mesh-info-omits-bbox`
(the ergonomic ticket that put `boundingBox` on `get_mesh_info`). The feature is
correct for non-empty meshes; it just has no guard for the empty/degenerate box.

## Verbatim repro (replay-confirmed 3x, isolated)

- `geometry.create_procedural_mesh {"actorName":"OracleEmptyMesh"}` → ok
  (`{"name":"OracleEmptyMesh","class":"DynamicMeshActor","enableCollision":false}`) — an empty 0-vertex mesh.
- `geometry.get_mesh_info {"actorName":"OracleEmptyMesh"}` → **client error**
  (reproduced three separate times, once in isolation):
  `Editor not reachable at http://127.0.0.1:24966/mcp - launch the Unreal editor and retry. (Expecting value: line 43 column 11 (char 1527))`
- Contrast — the same verb on a **non-empty** box returns valid JSON:
  `geometry.get_mesh_info {"actorName":"OracleWallUV"}` (8v/12t box) →
  `{"actorName":"OracleWallUV","vertexCount":8,"triangleCount":12,...,"boundingBox":{"min":{"x":-200,"y":-15,"z":-150},"max":{"x":200,"y":15,"z":150},"origin":{"x":0,"y":0,"z":0},"extent":{"x":200,"y":15,"z":150}},"message":"Mesh info retrieved"}`

## Guilty source (verbatim)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/MeshInfoHandler.cpp`
serializes the box unconditionally, with no empty/invalid guard:

- L116: `const FBox MeshBounds = UGeometryScriptLibrary_MeshQueryFunctions::GetMeshBoundingBox(Target.Mesh);`
- L118: `const FVector BoundsExtent = MeshBounds.GetExtent();`  ← `(Max-Min)*0.5` = `-inf` for an empty box
- L121-124:
  ```cpp
  BBoxObj->SetObjectField(TEXT("min"), JsonBuilders::BuildVectorJson(MeshBounds.Min));
  BBoxObj->SetObjectField(TEXT("max"), JsonBuilders::BuildVectorJson(MeshBounds.Max));
  BBoxObj->SetObjectField(TEXT("origin"), JsonBuilders::BuildVectorJson(BoundsOrigin));
  BBoxObj->SetObjectField(TEXT("extent"), JsonBuilders::BuildVectorJson(BoundsExtent));
  ```

`Plugins/PinWright/Source/PinWright/Private/Utils/JsonBuilders.h` writes the raw
doubles with no finite sanitization (L24-31):

```cpp
inline TSharedPtr<FJsonObject> BuildVectorJson(const FVector& Value)
{
    TSharedPtr<FJsonObject> Obj = MakeShared<FJsonObject>();
    Obj->SetNumberField(TEXT("x"), Value.X);
    Obj->SetNumberField(TEXT("y"), Value.Y);
    Obj->SetNumberField(TEXT("z"), Value.Z);
    return Obj;
}
```

severity rationale: impact=malformed/non-JSON response mis-reported as editor-down (High class) × reach=rare trigger (only empty/degenerate meshes, though get_mesh_info itself is every-session) -> Medium

**Fix:** two layers, either or both:
1. **In `get_mesh_info` (targeted):** guard the empty/invalid box — when
   `VertexCount == 0` or `!MeshBounds.IsValid`, emit a zeroed box (all
   `{x:0,y:0,z:0}`) or set `boundingBox: null` plus an `empty: true` flag,
   instead of serializing the inverted sentinel box.
2. **In `BuildVectorJson` / the JSON layer (systemic, preferred as a safety
   net):** sanitize non-finite doubles before serialization — replace
   `NaN`/`±Inf` with `0` (or `null`) so no handler can ever emit invalid JSON.
   Any handler that serializes a degenerate `FBox`/`FVector` is exposed to the
   same wire-corruption; a central finite-guard closes the whole class.

## History
- `#1-initial-repro` `OPEN` reporter — Found while adversarially probing `geometry.transform_uvs` (seed) in a modular-kit UV/bake task. Replay-confirmed 3x (once isolated): `geometry.create_procedural_mesh {OracleEmptyMesh}` then `geometry.get_mesh_info {OracleEmptyMesh}` returns a response the MCP client cannot parse — `Editor not reachable ... (Expecting value: line 43 column 11 (char 1527))` — while `get_mesh_info` on a non-empty box in the same session returns valid JSON with a finite `boundingBox`. Root cause: empty mesh → `GetMeshBoundingBox` returns the inverted-empty `FBox` (Min=+DBL_MAX, Max=-DBL_MAX) → `GetExtent()`=`(Max-Min)*0.5` overflows to `-inf` → `BuildVectorJson` (JsonBuilders.h:24-31) serializes the non-finite double via `SetNumberField` with no finite guard → UE's JSON writer emits a bare `inf`/`nan` token → invalid JSON → client rejects the whole response and mis-reports it as editor-down. Guilty lines: MeshInfoHandler.cpp:116-124 (unconditional bbox serialization, no empty/invalid guard) + JsonBuilders.h:24-31 (no non-finite sanitization). Distinct from `E-geometry-mesh-info-omits-bbox` (the ergonomic ticket that ADDED this bbox — correct for non-empty meshes) and from `B-connect-cue-nodes-crash` (a real socket-close crash, WinError 10054, not a JSON parse error). Fix: guard the empty box in `get_mesh_info` and/or sanitize non-finite doubles in `BuildVectorJson`.
