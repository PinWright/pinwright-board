---
id: E-geometry-create-name-vs-actorname
title: "geometry.create_* declare the actor slot as 'name'; every other geometry verb requires 'actorName' — no aliases, so a build pays two UNKNOWN_PARAMS/MISSING_PARAM round-trips"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [geometry, param-alias, name, actorname, create_procedural_mesh, dynamic-mesh, drift]
---

# The `geometry.create_*` verbs name the actor slot `name`, but every operate verb wants `actorName`

Same class of friction as `E-blueprint-param-name-path-vs-assetpath` (DONE),
`E-level-create-name-path-alias` (OPEN), `E-widget-asset-path-alias-drift`
(DONE), and `E-material-editor-param-name-drift` (DONE), in the
previously-untouched `geometry.*` actor-name slot.

The whole `geometry.create_*` family declares the new-actor slot as **`name`**
(an unaliased `RPC_PARAM_OPT`): `geometry.create_procedural_mesh`
(PrimitiveHandler.cpp:768-777, `RPC_PARAM_OPT("name", ...)` →
`Ctx.GetString(TEXT("name"), TEXT("ProceduralMesh"))`), and identically
`create_box` (:88,100), `create_sphere` (:142,150), `create_cylinder`
(:181,...). But **every** verb that subsequently operates on that same actor
requires **`actorName`** (`RPC_PARAM_REQ("actorName", ...)`, no alias):
`geometry.get_mesh_info`, `geometry.append_triangle`, `geometry.append_vertex`,
`geometry.recalculate_normals`, `geometry.generate_collision`,
`geometry.convert_to_static_mesh`, and the rest of the namespace
(MeshInfoHandler.cpp:86-88 is representative — `RPC_PARAM_REQ("actorName",
"string", "Name of the DynamicMeshActor")`).

So within a single procedural-mesh build the caller must spell the **same
actor's name** two different ways: `name` to create it, then `actorName` for
every following call. CLAUDE.md's "camelCase and snake_case aliases" rule does
not cover this — `name` and `actorName` are distinct names, not casing
variants — and neither slot carries the other as an alias, so guessing wrong in
either direction is a hard error, not a silent accept.

This is the same "create"-verb destination-slot pattern that
`E-material-editor-param-name-drift #2` deliberately left UNALIASED for
`create_material` (`name`+`path`) and that `E-level-create-name-path-alias`
flags for `level.create` (`levelName`+`levelPath`). The `geometry` case is a
cleaner argument for aliasing than either, because here the create verb and the
operate verbs live in the **same namespace** and operate on the **same actor
within one workflow** — the spelling flip happens mid-build, not just between
unrelated namespaces.

## Repro (verbatim, replayed live this session)

1. `geometry.create_procedural_mesh {actorName:"CrystalShard", location:{...}, enableCollision:true}`
   → `[UNKNOWN_PARAMS] Unknown parameter(s) for 'geometry.create_procedural_mesh': [actorName]. Valid parameters: [name, location, rotation, scale, enableCollision].`
   Retry with `{name:"CrystalShard", ...}` → succeeds.
2. `geometry.get_mesh_info {name:"CrystalShard"}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName' (type: string)`
   Retry with `{actorName:"CrystalShard"}` → `{vertexCount:0, triangleCount:0, ...}` succeeds.

Both errors are accurate (not misleading), and both retries succeed with zero
blocked progress — this is pure guessability/round-trip overhead, exactly the
shape of the four precedent param-drift tickets. The seed method itself behaves
correctly; only the slot naming diverges from its own namespace.

## What it should do

Mirror the dispatcher `FParamSpec` alias machinery from
`E-blueprint-param-name-path-vs-assetpath #4` (the one that finally made
alias-only required params validate at the wire level). Standardize the alias
set, not the canonical name, so existing callers keep working:
- Annotate the `geometry.create_*` `name` slot with an `actorName` alias (and
  accept `actorName` in the `Ctx.GetString(TEXT("name"), ...)` reads), so the
  create verb accepts the same spelling its siblings demand.
- Optionally annotate the operate verbs' `actorName` slot with a `name` alias
  for full symmetry. At minimum the create→operate direction (create accepting
  `actorName`) removes the mid-build flip, since once the actor exists every
  remaining call already uses `actorName`.

The wiki pages are correct (`geometry-create-procedural-mesh.md` documents
`name`; the operate-verb pages document `actorName`), so this is a
guessability/alias gap, not a docs gap — reading both pages first avoids the
wasted calls, but the natural same-name reuse across one build is the friction.

## History
- `#1-initial-repro` `OPEN` reporter — Struggle audit of the
  `geometry.create_procedural_mesh` "CrystalShard" hand-authored-mesh task
  (build empty DynamicMeshActor → append_triangle ×6 → append_vertex →
  recalculate_normals → generate_collision → convert_to_static_mesh; outcome
  done, full sequence replayed clean this session). The attempt agent ate two
  param round-trips: `geometry.create_procedural_mesh {actorName:...}` →
  `[UNKNOWN_PARAMS] ... [actorName]. Valid parameters: [name, location,
  rotation, scale, enableCollision]`, then `geometry.get_mesh_info {name:...}` →
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName'`, each
  corrected on retry. Source: PrimitiveHandler.cpp:768-777 declares the create
  slot as `RPC_PARAM_OPT("name", ...)` (same for create_box :88 / create_sphere
  :142 / create_cylinder :181), while MeshInfoHandler.cpp:86-88 and the rest of
  the namespace declare `RPC_PARAM_REQ("actorName", ...)` with no alias.
  Sibling of the four DONE/OPEN path/param-alias-drift tickets, here in the
  `geometry.*` actor-name slot. Distinct from `E-geometry-auto-uv-redundant-with-unwrap-uv`
  (duplicate XAtlas verbs) and `E-geometry-deformer-echo-mesh-counts` (deformers
  omit count echoes); this is purely the `name` vs `actorName` slot drift.
  Fix: dispatcher `FParamSpec` alias from `E-blueprint-param-name-path-vs-assetpath #4`,
  aliasing the create-verb `name` slot to accept `actorName`.
- `#2-fix` `IN-REVIEW` developer — Aliased the `geometry.create_*` `name` slot to
  accept `actorName` (the spelling every operate verb in the namespace demands),
  removing the mid-build spelling flip. Canonical stays `name`, so existing
  callers are unaffected; the dispatcher already honors `FParamSpec.Aliases` for
  both required-param satisfaction and the known-params set (RpcDispatcher.cpp:34,
  :59). New geometry-local helper `Handlers/Geometry/GeometryNameParamUtils.h`
  (built on the generic `ParamAliasUtils::MakeAliasParamSpec`, mirroring the
  per-namespace BlueprintHandlerUtils/MaterialHandlerUtils pattern) supplies the
  `GEOMETRY_CREATE_NAME_PARAM(Desc)` spec macro and `ResolveCreateName(Ctx, Default)`
  reader. Applied to all 16 create verbs in `PrimitiveHandler.cpp` (create_box,
  create_sphere, create_cylinder, create_cone, create_capsule, create_torus,
  create_plane, create_disc, create_stairs, create_spiral_stairs, create_ring,
  create_arch, create_pipe, create_ramp, revolve, create_procedural_mesh): each
  `RPC_PARAM_OPT("name", ...)` → `GEOMETRY_CREATE_NAME_PARAM(...)` and each
  `Ctx.GetString(TEXT("name"), ...)` body read → `ResolveCreateName(...)` so the
  alias resolves end-to-end into the spawned actor's label. Took the
  create→operate direction the ticket's "What it should do" prioritizes (left the
  operate verbs' `actorName` slot as-is — once the actor exists every remaining
  call already uses `actorName`). Regression test
  `Tests/Geometry/TestGeometryCreateNameParamAlias.cpp`: (1) static — every
  create verb's `name` spec carries the `actorName` alias; (2) end-to-end — the
  real dispatcher accepts `geometry.create_procedural_mesh {actorName:...}`
  without UNKNOWN_PARAMS and the spawned actor's label resolves from the alias
  value. Reverting the alias fails both layers.
