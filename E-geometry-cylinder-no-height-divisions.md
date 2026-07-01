---
id: E-geometry-cylinder-no-height-divisions
title: "geometry.create_cylinder / create_cone hardcode AppendCylinder/Cone height-steps to 1 and expose no height-division param, so axis deformers (twist/bend/taper) can't read along the primitive — while sibling create_pipe already exposes heightSteps"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [geometry, create_cylinder, create_cone, height-steps, heightSteps, segments, twist, bend, taper, deformer, intra-namespace-inconsistency]
---

# `create_cylinder`/`create_cone` give no way to subdivide along the axis, defeating any twist/bend/taper that should read along the primitive's length

`geometry.create_cylinder` exposes only `radius`, `height`, and `segments`
(radial), then calls `AppendCylinder(..., Segments, 1, true, ...)` with the
**height-steps argument hardcoded to `1`** (PrimitiveHandler.cpp:200-202).
`geometry.create_cone` is identical: `AppendCone(..., Segments, 1, true, ...)`
(PrimitiveHandler.cpp:241-243). With a single height ring, the side wall has no
intermediate loops, so an axis deformer applied afterward
(`geometry.twist`/`bend`/`taper`) can only reposition the top and bottom rings —
it **cannot** produce a spiral/curve that reads along the body, because there
are no intermediate vertices to displace. The requested deform silently
under-delivers; the mesh stays low-poly straight-walled.

This is an **intra-namespace inconsistency**, not a hard API limit. The sibling
`geometry.create_pipe` in the *same file* already exposes the exact knob:
`RPC_PARAM_OPT("heightSteps", "number", "Height steps (default 1)")`
(PrimitiveHandler.cpp:604) feeding `AppendCylinder(..., RadialSteps, HeightSteps,
...)` (:620-621). The underlying `UGeometryScriptLibrary_MeshPrimitiveFunctions::
AppendCylinder`/`AppendCone` both take a height-steps parameter natively — the
cylinder/cone handlers just pin it to `1` and never surface it. So the
capability exists one method over; cylinder and cone simply don't wire it
through.

## What it should do

Add a `heightSteps` (or `heightDivisions`) optional param to
`geometry.create_cylinder` and `geometry.create_cone`, defaulting to `1` for
backward compatibility, and pass it as the height-steps argument to
`AppendCylinder`/`AppendCone` in place of the literal `1`. Match the name and
default already used by `create_pipe` (`heightSteps`, default 1) for
cross-method consistency. This lets a column/pillar/horn build request enough
loops for a downstream twist/taper to read smoothly, which is the single most
common reason to subdivide a primitive's side wall.

**Workaround:** none that stays inside `create_cylinder` — to get axial loops the
caller must `create_pipe` (always hollow, needs an inner radius) or build the
primitive from `subdivide`/manual appends, or accept that the twist only moves
the end caps. The task here just accepted the under-divided result.

## Friction evidence (this task — geometry.twist "TempleColumn_01", 11 calls, outcome clean)

The story (step 1) explicitly asked for a cylinder with "enough radial segments
(about 24) **and height divisions** that a twist will read nicely along its
length," then (step 3) a 120° twist "so it spirals along the vertical axis." The
attempt agent created the cylinder with `radius=40, height=300, segments=24`,
got the requested radial detail, but **could not honor the height-divisions
request at all** — `create_cylinder` has no such param and hardcodes 1 height
ring. Per the self-report the twist therefore "only redistributes the top/bottom
rings rather than spiraling through intermediate loops." The counts stayed
74v/144t across baseline→twist→taper (a 24-segment cylinder with a single height
ring), confirming no intermediate loops existed for the deformer to spiral. The
build still completed and baked (`/Game/TempleProps/SM_TempleColumn_01`,
outcome `clean`), so this is a PROCESS/capability gap — the explicit user intent
was unachievable through the API, not an error the agent could retry around.
The friction note: "create_cylinder exposes no height-divisions param and the
handler hardcodes 1 height subdivision (AppendCylinder ...,Segments,1,...) ... the
requested 'enough height divisions' is not achievable via the API."

Distinct from `E-geometry-create-name-vs-actorname` (the `name` vs `actorName`
slot drift) and `E-geometry-deformer-echo-mesh-counts` (deformers omit count
echoes): this is a missing primitive-tessellation param that makes the
*downstream deformer* a no-op along the axis, evidenced live by the flat
74v/144t through a twist+taper chain.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.twist` "TempleColumn_01" twisted-column task (11 calls, outcome clean, judge filed nothing). PROCESS/capability finding: `create_cylinder` (PrimitiveHandler.cpp:200-202, `AppendCylinder(...,Segments,1,...)`) and `create_cone` (:241-243, `AppendCone(...,Segments,1,...)`) hardcode height-steps to 1 and expose no height-division param, so the story's explicit "enough height divisions that a twist will read nicely along its length" was unachievable — the 120° twist could only move the end rings (counts stayed 74v/144t baseline→twist→taper). Sibling `create_pipe` already exposes `heightSteps` (PrimitiveHandler.cpp:604,620-621) over the same native `AppendCylinder` height-steps arg, so the capability is wired one method over. Fix: add a `heightSteps` optional param (default 1) to `create_cylinder`/`create_cone`, matching `create_pipe`. Workaround: none inside `create_cylinder`.
- `#2-fix` `IN-REVIEW` developer — Added an optional `heightSteps` param (default 1, backward-compatible) to `geometry.create_cylinder` and `geometry.create_cone`, mirroring the existing sibling `create_pipe` verbatim (`RPC_PARAM_OPT("heightSteps", ...)` + `int32 HeightSteps = Ctx.GetInt(TEXT("heightSteps"), 1)`), and replaced the literal `1` height-steps argument in the `AppendCylinder`/`AppendCone` calls with `HeightSteps`. Now an axis deformer (`twist`/`bend`/`taper`) has intermediate side-wall loops to displace. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Geometry/PrimitiveHandler.cpp` (create_cylinder schema+call, create_cone schema+call). Regression tests added to `Source/EditorAutomationRpcGateway/Private/Tests/World/TestGeometryHandlers.cpp` — `geometry.create_cylinder.HeightStepsSubdivides` and `geometry.create_cone.HeightStepsSubdivides` build a primitive via the production handler with `heightSteps=1` vs `heightSteps=4`, read back the spawned `ADynamicMeshActor`'s actual `UDynamicMesh::GetTriangleCount()`, and assert the subdivided build has strictly more triangles; reverting the fix (dropping the param / restoring the literal 1) makes both counts identical and fails the assertion.
