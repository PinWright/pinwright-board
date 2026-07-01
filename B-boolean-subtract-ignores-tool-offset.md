---
id: B-boolean-subtract-ignores-tool-offset
title: "geometry.create_* double-applies the spawn transform: location/rotation/scale baked into BOTH mesh vertices and actor transform mis-places every non-origin primitive (visible as boolean_subtract silently no-op'ing an offset cutter)"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, primitive, transform, double-apply, boolean, silent-noop]
---

# `geometry.create_*` double-applies the spawn transform

Every `geometry.create_*` primitive handler (create_box, create_sphere,
create_cylinder, create_cone, create_capsule, create_torus, create_plane,
create_disc, create_stairs, create_spiral_stairs, create_ring, create_arch,
create_pipe, create_ramp, revolve, …) reads ONE `FTransform` from the
location/rotation/scale payload and feeds that SAME transform to **both**:
1. the `Append*` Geometry-Script primitive call — which bakes the
   location/rotation/scale into the mesh's **local vertices**, and
2. `SpawnDynamicMeshActor` — which sets the same location/rotation on the actor
   and calls `SetActorScale3D`.

So the spawn transform is **applied twice**. A primitive requested at location
`+80` lands its mesh-local vertices centered at `+80` *and* its actor at `+80`,
so its true world position is `+160` (and a requested scale is squared, a
requested rotation doubled). Any `create_*` with a non-identity
location/rotation/scale is mis-placed by 2x.

## Most visible victim: `boolean_subtract` silently no-ops an offset cutter

The canonical "place a cutter where you want the hole, then subtract" workflow
(carve a doorway, drill a bolt hole) breaks. The cutter is created at an offset
so it overlaps the target — but because its mesh is *also* pre-offset, its true
world footprint is shifted to 2x and no longer overlaps the target. The engine's
`ApplyMeshBoolean` correctly finds zero intersection and returns the target mesh
**unchanged**; the handler reports `success:true` with
`resultTriangles == targetTriangles`. No error, no warning: a **silent
success-with-no-effect**. The caller is told it succeeded, so it proceeds to
bevel / collision / convert_to_static_mesh on geometry that was never cut.

## Repro (verbatim, replay-confirmed against mcp__editor-automation__call)

Identical tool geometry, identical target, the **only** difference is the
tool actor's `location` — yet one cuts and one silently no-ops.

Setup:
```
geometry.create_box {name:"RTarget",      width:200, height:200, depth:200, location:{x:0,y:0,z:0}}
geometry.create_box {name:"RToolOffset",  width:60,  height:60,  depth:400, location:{x:80,y:0,z:0}}
geometry.create_box {name:"RToolOrigin",  width:60,  height:60,  depth:400, location:{x:0,y:0,z:0}}
```
RTarget occupies world X/Y/Z [-100,100]. RToolOrigin is a Z-rod through the
center. RToolOffset is the SAME rod shifted +80 in X — world X [50,110], so it
still overlaps RTarget across X [50,100] with full Y/Z overlap (a 50x60x200
solid intersection that must carve a notch).

Offset tool — NO CUT:
```
geometry.boolean_subtract {targetActor:"RTarget", toolActor:"RToolOffset", keepTool:true}
-> {"targetActor":"RTarget","operation":"Subtract","success":true,
    "targetTriangles":12,"toolTriangles":12,"resultTriangles":12}
geometry.get_mesh_info {actorName:"RTarget"}  -> 8 verts / 12 tris  (UNCHANGED)
```

Same target, tool at origin — CUTS:
```
geometry.boolean_subtract {targetActor:"RTarget", toolActor:"RToolOrigin", keepTool:true}
-> {"targetActor":"RTarget","operation":"Subtract","success":true,
    "targetTriangles":12,"toolTriangles":12,"resultTriangles":76}
```

Additional data points from the same replay session that bracket the threshold:
- target@(500,0,0) + tool@(500,0,0) (coincident) -> cuts (12 -> 76).
- target@(500,0,0) + tool@(520,0,0) (offset +20) -> cuts (12 -> 82).
- target@(0,0,0)   + tool@(80,0,0)  (offset +80) -> NO cut (12 -> 12).
- target@(0,0,0)   + tool@(100,100,100), 100^3 corner overlap -> NO cut (12 -> 12).
So small relative offsets cut; larger relative offsets (still clearly
overlapping) silently produce no geometry change.

## Root cause (the boolean is NOT at fault)

The boolean dispatch is correct. `HandleBooleanOperationImpl`
(`BooleanHandler.cpp:121-129`) passes both actors' world transforms
(`Target.Actor->GetActorTransform()`, `Tool.Actor->GetActorTransform()`) into
`ApplyMeshBoolean`, and the engine resolves both meshes into a common space and
returns the result in the target's local space. Given the inputs it is handed,
it produces the right answer.

The defect is upstream, in the **primitive creators**. `create_box`
(`PrimitiveHandler.cpp:104-127`) builds one `FTransform` via
`GeometryUtils::ReadTransformFromPayload` (`GeometryUtils.cpp:82-93`) and passes
it to BOTH `AppendBox` (line 121 — bakes the offset into mesh-local vertices) AND
`SpawnDynamicMeshActor` (line 127 — sets the same offset on the actor). The
offset is double-applied. So the two actors do **NOT** actually overlap in world
space: the offset tool's mesh vertices carry the `+80`, and its actor carries it
again, putting its real world footprint at `+130..+190` — disjoint from the
target's `-100..+100`. The boolean is right to produce no cut. The same pattern
is present in every `create_*` handler (`PrimitiveHandler.cpp` lines 121, 163,
206, 249, 291/296, 338, 379, 415, 457, 501, 539, 584, 628/634, 685, 758).

The handler additionally reports `success` purely on `ResultMesh != nullptr`
(`BooleanHandler.cpp:131,168,176`), which is true even when the engine returns
the target unchanged — so a genuine no-op is indistinguishable from a real cut
except by diffing `resultTriangles` against `targetTriangles`.

## What it should do
Each `create_*` should build the mesh in **local space** (`Append*` receives
`FTransform::Identity`) and apply the requested location/rotation/scale **only**
on the spawned actor's world transform — the single source of truth that the
boolean and the downstream array/transform/collision handlers already assume. As
a secondary safety-net for the genuinely-disjoint case, a boolean whose result
triangle count equals the target's should surface that (a `changed:false` flag)
so the caller is not misled into thinking the cut happened.

**Workaround:** position the cutter with `geometry.translate_mesh` (which moves
mesh-local geometry, not the actor) instead of passing `location` on
`create_box`; or independently diff `resultTriangles` against `targetTriangles`
after every boolean. Neither is obvious.

**Fix:** in `PrimitiveHandler.cpp`, pass `FTransform::Identity` to every
`Append*` primitive call (and to `create_pipe`'s two cylinder builds), keeping
the world `FTransform` only on `SpawnDynamicMeshActor`. Add a `changed` flag to
the boolean response in
`HandleBooleanOperationImpl` (`BooleanHandler.cpp`) computed as
`resultTriangles != targetTriangles`.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed against
  mcp__editor-automation__call: subtracting an offset-but-overlapping tool
  (RToolOffset @ x=80) from RTarget @ origin returns success:true with
  resultTriangles==targetTriangles==12 and leaves the mesh at 8v/12t; the
  identical tool at the origin (RToolOrigin) cuts 12->76. The only difference
  is the tool actor's location, so an overlapping offset cutter is silently
  dropped. Isolated A/B above; offsets of +20 still cut while +80 does not.
- `#2-rescope-and-fix` `IN-REVIEW` developer — Rescoped: the symptom is real and
  reproduced, but the root cause is NOT the boolean (its `ApplyMeshBoolean`
  transform handling is correct). It is a double-applied spawn transform in the
  `create_*` primitives — one `FTransform` fed to both the `Append*` call (bakes
  offset into mesh-local verts) and `SpawnDynamicMeshActor` (sets it on the
  actor), so an offset cutter's true world footprint is shifted to 2x and no
  longer overlaps the target. Verified live via `get_vertex_position` (local
  x=50 for the offset tool vs -30 for the origin tool, delta = the +80) +
  `actor.describe` (actor also at +80). Fix: `PrimitiveHandler.cpp` now passes
  `FTransform::Identity` to every `Append*` call (box/sphere/cylinder/cone/
  capsule[both UE branches]/torus/plane/disc/stairs/spiral_stairs/ring/arch/
  pipe[both cylinders]/ramp/revolve), making the actor transform the single
  source of truth. Secondary hardening: `BooleanHandler.cpp`'s
  `HandleBooleanOperationImpl` now emits a `changed` flag
  (`resultTriangles != targetTriangles`) so a genuinely-disjoint no-op is
  reportable. Files: `Handlers/Geometry/PrimitiveHandler.cpp`,
  `Handlers/Geometry/BooleanHandler.cpp`. Regression tests added to
  `Private/Tests/World/TestGeometryHandlers.cpp`:
  `geometry.create_box.OffsetNotDoubleApplied` (asserts the spawned offset box's
  mesh-LOCAL bbox center stays at the origin and its WORLD center equals the
  requested +80, not +160) and `geometry.boolean_subtract.OffsetToolCuts`
  (offset overlapping cutter now yields resultTriangles > targetTriangles and
  changed=true) — both fail if the identity-transform fix is reverted.
