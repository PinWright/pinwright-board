---
id: E-geometry-primitive-orientation-axes-undocumented
title: "geometry.create_box width/height/depth axis mapping (W->X, H->Y, D->Z) and create_arch's torus plane (lies in X-Y, needs roll=-90 to stand apex-up) are undocumented, so placing/orienting a primitive for a boolean takes a create->probe->delete->recreate spiral plus a source dive"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [geometry, create_box, create_arch, orientation, axis, plane, roll, docs, wiki, discoverability, boolean]
---

# create_box's dimension->axis mapping and create_arch's build plane / apex orientation are undocumented

Two `geometry` primitive verbs build their mesh into a fixed local-axis layout
that the wiki and param descriptions never state, so a caller positioning a
primitive to overlap another (the prerequisite for any `boolean_subtract` /
`boolean_union`) has to discover the layout empirically:

1. **`geometry.create_box` dimension->axis mapping.** The verb takes
   `width` / `height` / `depth`, and the natural reading is that `height` is the
   vertical (Z) dimension. It is not: the handler calls
   `AppendBox(DynMesh, Options, Transform, Width, Height, Depth, ...)`
   (`PrimitiveHandler.cpp:117-121`), and Geometry Script's `AppendBox` maps
   **Width->X, Height->Y, Depth->Z**. So a "400 wide x 500 tall x 80 deep" wall
   authored as `{width:400, height:500, depth:80}` comes out as a flat slab lying
   in the X-Y plane (tall along Y, only 80 thick along Z) instead of an upright
   wall — to stand it up you must pass `{width:400, height:80, depth:500}`. The
   wiki entry (`wiki-generated/geometry.create_box.md:15-17`) and the param
   strings (`PrimitiveHandler.cpp:92-94`) say only "Box width/height/depth
   (default 100)" with no axis note.

2. **`geometry.create_arch` build plane and apex direction.** `create_arch` is a
   partial torus built with `AppendTorus(... EGeometryScriptPrimitiveOriginMode::Center ...)`
   (`PrimitiveHandler.cpp:576-578`), which revolves in the **X-Y plane about Z** —
   so the arch lies flat by default, and to stand it apex-up (a doorway opening)
   you must rotate it. The non-obvious part is the **sign**: `roll=-90` stands it
   apex-up, `roll=+90` points the apex **down** (below the wall). Nothing in the
   wiki (`wiki-generated/geometry.create_arch.md`) or the param strings states
   the build plane, that a rotation is needed to stand it upright, or which roll
   sign yields apex-up.

Both facts are pure discoverability gaps (the verbs behave correctly), but they
compound badly: the only way to confirm a primitive is positioned to overlap its
boolean partner is to read its actual geometry, and `geometry.get_vertex_position`
returns **mesh-LOCAL** coordinates (it ignores the actor's world location /
transform), which the wiki also does not state — so the agent could not read
world-space overlap directly and had to probe transform semantics (move the actor,
re-sample, confirm the vertex didn't move) before it could reason about the cut.

This is the same docs-gap shape already filed as `E-geometry-warp-extent-semantics`
(the warp deformers' `extent` units/symmetry undocumented, forcing a source dive)
and `E-geometry-cylinder-no-height-divisions` — a `geometry` primitive whose
spatial convention is only discoverable by reading the C++. Distinct from
`B-boolean-subtract-ignores-tool-offset` (the *boolean* silently no-ops on an
offset tool — a real bug the judge filed): even with that bug fixed, a caller
still cannot author an upright wall or apex-up arch without these orientation
facts. Distinct from `E-geometry-create-name-vs-actorname` (the `name` vs
`actorName` slot drift) and `E-geometry-mesh-info-omits-bbox` (the read verb
omits the bbox).

## What it should do (docs only — works, just under-documented)

Improve the **`docs/wiki-src/geometry.md`** overlay (currently a 5-line namespace
blurb with no per-method param notes) with a short "primitive orientation"
section, plus optionally enrich the handler param strings:

- **create_box:** state that `width`->X, `height`->Y, `depth`->Z (local axes,
  origin-centered). To make an upright wall of footprint WxD and vertical height
  H, pass `{width:W, height:D, depth:H}` (i.e. the *vertical* dimension is
  `depth`, not `height`).
- **create_arch:** state that the arch is built in the local X-Y plane (flat by
  default) and that `rotation.roll = -90` stands it apex-up (`+90` points the
  apex down); the opening spans the `angle` sweep.
- **get_vertex_position:** note that the returned coordinates are mesh-LOCAL
  (the actor's world transform is not applied), so to reason about world-space
  overlap of two actors, combine each with its `actor.describe` transform (or
  read `actor.get_bounding_box`, which is world-space).

One paragraph in the overlay removes the entire empirical-discovery spiral for
every prop-build that positions one primitive to cut/union another.

## Friction evidence (this task — geometry "stone gateway" prop, ~65 calls, outcome tool_bug)

The build was: box wall -> arch cutter -> `boolean_subtract` doorway -> bevel ->
recalculate_normals -> get_mesh_info -> generate_collision -> convert_to_static_mesh.
The bulk of the ~65 calls was orientation discovery, not the eight intended steps:

- **create_box axis:** the first `GatewayWall` (`{width:400, height:500, depth:80}`)
  came out a flat slab; the agent only learned `width->X, height->Y, depth->Z`
  after `get_vertex_position` sampling, deleted it, and recreated it upright as
  `400/80/500`. Friction note: *"create_box maps width->X, height->Y, depth->Z,
  so my intended upright wall came out as a flat slab; nothing in the wiki states
  the axis mapping."*
- **create_arch roll sign:** the agent created the arch three times to find the
  apex-up orientation — `roll=90` (apex down), then probing, then `roll=-90`
  (apex up confirmed). Friction note: *"create_arch (AppendTorus) lies in the X-Y
  plane and needs roll=-90 (NOT +90) to stand apex-up — I had the roll sign
  backwards and the apex pointed down, below the wall."*
- **get_vertex_position local coords:** *"geometry.get_vertex_position returns
  mesh-LOCAL coords (ignores the actor's location), so I couldn't read world
  overlap directly and had to probe transform semantics"* — driving a string of
  `actor.set_transform` + `get_vertex_position` + `actor.describe` probe calls
  (move to z=-260, re-sample, move to z=1000, re-sample, move back).
- **Source dive (last resort):** *"read PrimitiveHandler.cpp/BooleanHandler.cpp
  (plugin source, last resort) to learn AppendTorus's plane and that the boolean
  correctly uses actor world transforms."*

Call-log counts: `geometry.create_box` x3 (slab, sanity-test, final upright),
`geometry.create_arch` x4 (no-rot, roll=90 down, roll=-90 up, final),
`actor.delete` x6 (discarded wrong-axis/wrong-apex actors), `get_vertex_position`
x8 (orientation/local-coord probes), `actor.set_transform` x5 and `actor.describe`
x3 (transform-semantics probes). The `B-boolean-subtract-ignores-tool-offset` bug
the judge filed sat *on top of* this confusion — the agent had to first rule out
"is my arch even positioned right?" (an orientation question) before it could
isolate the offset-tool boolean bug via a box-box sanity test. A one-paragraph
orientation note in the overlay collapses all of that into a single correct
create call per primitive.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the geometry "stone
  gateway" prop task (~65 calls, outcome tool_bug; judge filed
  `B-boolean-subtract-ignores-tool-offset` for the offset-tool boolean no-op).
  Distinct PROCESS/docs finding: `geometry.create_box`'s dimension->axis mapping
  (width->X, height->Y, depth->Z; `PrimitiveHandler.cpp:117-121`,
  `AppendBox(...Width,Height,Depth...)`) and `geometry.create_arch`'s torus build
  plane (lies in local X-Y, `roll=-90` stands it apex-up, `+90` apex-down;
  `PrimitiveHandler.cpp:576-578`, `AppendTorus`) are undocumented in both the
  wiki (`geometry.create_box.md`/`geometry.create_arch.md`) and the param
  strings, and `get_vertex_position` silently returns mesh-LOCAL coords. The
  agent ate create_box x3, create_arch x4, actor.delete x6, get_vertex_position
  x8, plus ~8 set_transform/describe transform-semantics probes, and ultimately a
  source dive to learn the orientation conventions — all before it could even
  reason about the boolean cut. Same docs-gap shape as
  `E-geometry-warp-extent-semantics` (source dive for an undocumented spatial
  convention). Fix (downstream docs): add a "primitive orientation" note to the
  `docs/wiki-src/geometry.md` overlay covering create_box axis mapping,
  create_arch plane/roll-sign, and get_vertex_position local-space semantics.
- `#2-orientation-overlay` `IN-REVIEW` developer — Added the undocumented spatial
  conventions to the `geometry` wiki overlay so a caller never has to discover them
  empirically. `Docs/wiki-src/geometry.md` now carries a `## Primitive orientation`
  namespace section plus three `### method` H3 sections: **create_box** (width->X,
  height->Y, depth->Z; the *vertical* extent is `depth`, not `height`; upright-wall
  recipe `{width:W, height:D, depth:H}`), **create_arch** (the torus revolves in the
  local X-Y plane so the arch lies flat; `rotation.roll = -90` stands it apex-up,
  `+90` points the apex down; opening spans `angle`), and **get_vertex_position**
  (returns mesh-LOCAL coords — the actor world transform is not applied, so
  `actor.set_transform` doesn't change the readback; reason about world overlap via
  `actor.describe` transform or world-space `actor.get_bounding_box`). The conventions
  verified against current source: `AppendBox(... Width, Height, Depth ...)`
  (`PrimitiveHandler.cpp:133-137`), `AppendTorus(... revolve X-Y ...)`
  (`PrimitiveHandler.cpp:596-598`), and `GetVertexPosition(Target.Mesh, ...)` with no
  actor transform applied (`MeshInfoHandler.cpp:136-151`); the note is authored against
  post-`B-boolean-subtract-ignores-tool-offset` single-apply semantics (Append* is
  called at `FTransform::Identity`, so the axis mapping/build plane are
  transform-independent and the local-vs-world divergence is cleaner). Regression test:
  `Tests/Infra/TestGeometryPrimitiveOrientationDocs.cpp` — four
  `IMPLEMENT_SIMPLE_AUTOMATION_TEST`s drive the live `WikiHandler::RenderPage` path for
  the `geometry` namespace page and the three method pages, asserting on
  overlay-exclusive markers (axis mapping, the `-90` roll sign, mesh-LOCAL caveat) that
  fail iff the overlay sections are reverted. Files: `Docs/wiki-src/geometry.md`,
  `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestGeometryPrimitiveOrientationDocs.cpp`.
