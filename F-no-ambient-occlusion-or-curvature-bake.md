---
id: F-no-ambient-occlusion-or-curvature-bake
title: "No verb anywhere bakes ambient occlusion, cavity, curvature or thickness — not into vertex colours, not into a texture — so a mesh built from interpenetrating .pwmodel parts has no contact shadow at any part junction and no way to author one"
status: OPEN
severity: Medium
category: feature
tags: [geometry, texture, static_mesh, pwmodel, ambient-occlusion, cavity, curvature, thickness, bake, vertex-color, contact-shadow, missing-capability]
encounters: 3
lastSeen: 2026-09-06T07:00:00Z
---

# The plugin can build the mesh but cannot shade where its parts meet

## The gap

There is no verb in the PinWright RPC surface, and no `.pwmodel` op, that computes an occlusion,
cavity, curvature or thickness term from geometry and writes it anywhere — vertex colours, a
texture, or any other consumer. A mesh can be modelled, UV'd, collided, LOD'd and baked to a
`StaticMesh` entirely through the plugin, and then there is nothing to author the one signal that
makes a multi-part mesh read as solid.

## The sweep that establishes it

Exhaustive read of `Saved/PinWright/wiki/`, all namespaces, 2026-09-05:

- `lighting.set_ambient_occlusion` is the **only** verb in any namespace with `occlusion` in its
  name that concerns rendering, and it is a screen-space post-process setting on a
  `PostProcessVolume` (`enabled`, `intensity`, `radius`). It writes nothing to any mesh, is
  per-level rather than per-asset, and cannot resolve a millimetre-scale part junction.
  (`performance.configure_occlusion_culling` and `audio.authoring.configure_occlusion` are
  visibility culling and sound propagation — unrelated.)
- `geometry.*` — 91 verb pages. No occlusion, cavity, curvature or thickness computation of any
  kind. `check_health`, `get_mesh_info` and `measure` are read-only and return no occlusion field.
  The closest thing is `geometry.pack_uv_islands`, whose own page frames its resolution parameter
  as "the resolution of the texture you intend to bake to"
  (`Saved/PinWright/wiki/geometry.pack_uv_islands.md:25`) — it packs UVs **for** a bake the plugin
  cannot then perform.
- `texture.*` — 25 verb pages, a pure 2D image pipeline with **no geometry input at all**. Nothing
  in it takes a mesh, an actor, or a UV set.
- `static_mesh.*` is four verbs: `bake_transform`, `describe`, `set_collision_complexity`,
  `set_material`. `bake_transform` bakes a transform, not a signal.
- `grep -i` for `vertex_paint`, `mesh paint`, `dirty vertex` across the whole wiki: **zero hits**.

## Why a tiling material cannot substitute

A tiling detail or height map is a function of UV, so it cannot know where two solids meet. Any
mesh authored through `.pwmodel` as interpenetrating parts — which is the format's core idiom, and
what its boolean and merge ops exist to serve — therefore has no contact shadow at any junction,
and no verb can produce one. This is not an aesthetic nicety: the darkening at a junction is the
cue that tells a viewer two parts are one object rather than two shapes sharing a bounding box.

Concrete case that produced this ticket, on `Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel` and
`Content/FPS/Weapons/Meshes/SM_WPN_Pistol.pwmodel`: the AR has a receiver/handguard seam at
x 13.50..14.00, and the pistol carries a 110-stud stipple field (11x5x2 mirrored, each stud 0.26
square standing 0.10 proud). Nine junctions were measured dark-free. A critic shown the render
recorded the stipple as "still a perfect 11x5 grid, and invisible without AO" — the field is
geometrically present and visually absent, because nothing shades the crevices between the studs.

## What is asked for

`geometry.bake_ambient_occlusion` (and the matching `.pwmodel` modifier `bake_ao`, so it runs
inside a model source rather than as a post-step), operating on the accumulated mesh: ray-cast the
mesh against **itself** from each vertex over a cosine-weighted hemisphere and write the result
into a chosen vertex-colour channel.

Parameters that make it usable rather than merely present:

- `occlusionRadius` — world units. The load-bearing one: a part junction must occlude while the far
  side of the same weapon must not. Without a radius the result is a global bent-normal darkening
  that says nothing about local contact.
- `samples`, `bias` — quality and self-hit rejection.
- `channel` (`r|g|b|a`) — **critically, alpha alone, without disturbing RGB.** See the constraint
  below; this is what makes the feature consumable at all.
- `blend` (`replace|multiply`), `strength`.

A texture-space variant (`bake_ao_to_texture`, using UV0, writing a `UTexture2D`) would also be
useful and pairs naturally with `pack_uv_islands`, but it is secondary. The vertex/alpha bake is
the cheap one: it costs no extra interpolator, no second UV set and no texture memory, and it is
what a mesh of this class actually needs.

## The channel constraint is a hard blocker on consuming the result

Even given a bake, there is today no way to store it. `geometry.set_vertex_color` writes all four
components at once and `.pwmodel`'s `set_vertex_color` takes exactly `index`, `color`, `set_all`
(`Plugins/PinWright/Source/PinWrightGeometry/Private/Model/PwModelParser.cpp:1058-1062`, dispatched
at `PwModelCompiler.cpp:2920-2929`). On both weapon meshes RGB already carries a per-part albedo
ladder consumed by a `VertexTintAmount` material parameter, so writing alpha destroys the ladder.
Filed separately as `F-set-vertex-color-no-channel-mask`, because it is small, independently
fixable, and blocks this feature's only cheap output channel.

Related and also absent: `.pwmodel` has **no** op that sets colour by position, by falloff, by a
bounding region, by proximity to another part, or on a selected subset of vertices or faces. The
entire vertex-colour surface is `set_vertex_color` plus `color=` on 16 generators and `colors=` on
`append_buffers`. So even a hand-authored approximation of contact shading — darken everything
within N units of the seam plane — is not expressible.

## Workaround used, and why it is not an equivalent

The occlusion was authored analytically **in the material** instead: a screen-space signed
surface-divergence curvature term, `DDX`/`DDY` of `VertexNormalWS` differenced against `DDX`/`DDY`
of world position. It ships and it reads better than nothing, but it is:

- **view-dependent** — the term changes with camera angle, so the "shadow" swims;
- **resolution-dependent** — it is a screen-space derivative, so it changes with output resolution
  and with distance;
- **bevel-dependent** — it needs a chamfer to have anything to differentiate; a perfectly sharp
  90-degree junction between two boxes produces no signal at all, which is precisely the junction
  class this was needed for;
- **per-material, not per-mesh** — every material that wants it must reimplement it, and a mesh
  using a shared library material cannot have it.

It is a substitute for a bake, not an equivalent to one.

severity rationale: impact=a whole authoring signal is unreachable; the workaround is
view-dependent and fails on the exact geometry that needs it x reach=any `.pwmodel` mesh built from
interpenetrating parts, which is the format's core idiom -> Medium

## History
- `#1-filed` `OPEN` reporter — Filed from the FPS WEAPONS stream after an exhaustive namespace-by-namespace read of `Saved/PinWright/wiki/` found no occlusion/cavity/curvature/thickness bake anywhere: `lighting.set_ambient_occlusion` is a screen-space `PostProcessVolume` setting that touches no mesh; `geometry.*` (91 verbs) computes no such term and its read-only queries (`check_health`, `get_mesh_info`, `measure`) return no occlusion field; `geometry.pack_uv_islands` packs UVs *for* a bake the plugin cannot perform (`geometry.pack_uv_islands.md:25`); `texture.*` (25 verbs) takes no geometry input at all; `static_mesh.*` is four verbs, none of them a signal bake; and `vertex_paint` / `mesh paint` / `dirty vertex` return zero hits wiki-wide. Concrete cost on `SM_WPN_AR` (receiver/handguard seam at x 13.50..14.00) and `SM_WPN_Pistol` (110-stud stipple field, 11x5x2 mirrored, each stud 0.26 square standing 0.10 proud): nine junctions measured dark-free, and a critic recorded the stipple as "still a perfect 11x5 grid, and invisible without AO". A tiling detail/height map is a function of UV and structurally cannot know where two solids meet, so it is not a substitute. Ask: `geometry.bake_ambient_occlusion` plus a `.pwmodel` `bake_ao` modifier, ray-casting the accumulated mesh against itself over a cosine-weighted hemisphere, with `occlusionRadius` in world units (so a junction occludes and the far side of the weapon does not), `samples`, `bias`, `channel` (`r|g|b|a`), `blend`, `strength`; a UV0 texture-space variant second. Blocked on `F-set-vertex-color-no-channel-mask` for its cheapest output: `.pwmodel`'s `set_vertex_color` takes only `index`/`color`/`set_all` (`PwModelParser.cpp:1058-1062`, `PwModelCompiler.cpp:2920-2929`) and the RPC writes all four components, so alpha cannot be written without destroying the per-part RGB albedo ladder both weapon meshes carry for `VertexTintAmount`. Workaround shipped instead: an analytic screen-space signed surface-divergence curvature term in the material (`DDX`/`DDY` of `VertexNormalWS` against `DDX`/`DDY` of world position) — view-dependent, resolution-dependent, needs bevels to exist so it produces nothing on a sharp 90-degree junction, and per-material rather than per-mesh.

- `#2-seen` `OPEN` reporter — Independently re-derived on the WEAPONS build-05 stream rather than inherited: `ls Saved/PinWright/wiki/ | grep -iE 'bake|occlus|cavity|_ao|ambient'` over all **1496** wiki pages returns exactly seven files, and every one is unrelated — `audio.authoring.configure_occlusion`, `audio.create_ambient_sound`, `lighting.set_ambient_occlusion` (a `PostProcessVolume` SSAO setting; its own page says it only writes `intensity`/`radius` and cannot even guarantee the pass runs, because `r.AmbientOcclusionLevels 0` vetoes it), `performance.configure_occlusion_culling`, `sequencer.bake_control_space`, `sequencer.bake_to_controlrig`, `static_mesh.bake_transform`. A whole-wiki `grep -ril 'ambient occlusion'` hits only `index.md`, `lighting.md` and `lighting.set_ambient_occlusion.md`. Confirmed for the second consecutive round.
  **New information that changes the workaround, not the ask.** The `#1` workaround (screen-space `DDX`/`DDY` curvature) was measured by the round-04 critic and cannot darken a junction: on `SM_WPN_AR`, `split_normals split_angle=40` makes the normal piecewise CONSTANT, so its screen derivative is zero everywhere except the single quad straddling an edge — 4 luminance levels at the receiver/handguard seam, 0-2 at the upper/lower seam. Two of the other three terms in that material's `AmbientOcclusion` output are equally incapable, and the vertex-alpha term is provably dead because every `color=` alpha in both `.pwmodel` sources is 1 — which is this ticket's own `F-set-vertex-color-no-channel-mask` blocker showing up as a shipped no-op. **A real bake IS reachable, but only outside the RPC surface**: `python.execute` +
  `GeometryScript_AssetUtils.copy_mesh_from_static_mesh` + `GeometryScript_MeshQueries.get_triangle_positions` dumps the triangles, and the occlusion integral, the projection rasteriser and the PNG encoder then have to be written by hand in stdlib Python and brought back in through `asset.import`. That is roughly 700 lines of producer per project to replace one verb, it cannot be expressed inside a `.pwmodel` at all, and it has to be re-run by hand whenever the model source changes — so the ask stands unchanged and the severity understates it: every consumer of `.pwmodel`'s interpenetrating-parts idiom pays this, not just this weapon kit.

- `#3-seen` `OPEN` reporter — Third consecutive independent confirmation (WEAPONS build 06, UE 5.8, EAContentExamples58 HEAD). Sweep repeated over all 1496 wiki pages including the two namespaces `#2` did not name explicitly: `model.*` is 24 pages, all `.pwmodel` authoring documentation and `compile`/`validate`/`describe_ops`, with no bake op; `geometry.*` is 91 verbs with no occlusion, cavity, curvature or thickness computation. Symptom and ask unchanged and not restated. **What is new is that `#2`'s own stated workaround does not work either, and the reason generalises.** `#2` said a real bake "IS reachable outside the RPC surface" via `python.execute` + `GeometryScript_MeshQueries` + a hand-written occlusion integral. That producer was written (700 lines, cosine-weighted rays marched against a voxel shell of the same mesh, sampled per triangle corner) and it measured **255/255 — dead flat — at the very junction it was built for**, the AR receiver/handguard seam. Two structural reasons, and neither is a tuning mistake: (a) the ray origin must be biased off the surface to escape self-hit in a voxel shell, at `1.75 * voxel` = 0.1575 uu on this mesh, and the seam is a **0.10 uu step** — the ray starts above the occluder and no voxel size fixes it, because resolving 0.10 uu needs a 0.03 uu voxel and the AR's grid at 0.03 uu is 1.35 GB; (b) sampling per triangle CORNER and interpolating barycentrically cannot represent a line at all — the receiver flank is a handful of large triangles, so a seam crossing the middle of one is averaged out of existence before it is written. The route that does work is a different algorithm: rasterise an orthographic **height field** per projection band and integrate the horizon over it per texel, subtracting the surface's own tangent slope so curvature is not read as cavity (visible fraction of a cosine-weighted hemisphere above horizon angle `a` is exactly `cos^2(a) = 1/(1+slope^2)`). Same seam then measures 255 -> 227, the upper/lower receiver seam 254 -> 182, the rail underside -> 120, the magwell roof -> 166, the pistol's 110 stipple studs 255 <-> 166 (against the 4-7 levels the ray-march gave), the slide serrations 255 <-> 119. **The point for this ticket:** a caller who reaches for the sanctioned escape hatch and writes the textbook ray-marched AO gets a plausible, green-reporting, entirely flat result, and only a controlled per-junction luminance measurement catches it. That is a second full algorithm to get right per project on top of the triangle dump, the atlas rasteriser, the PNG encoder and the re-import — and none of it can live in a `.pwmodel`, so it must be re-run by hand on every model edit. It was: the mesh was retriangulated by a concurrent stream mid-task (22,652 -> 11,110 triangles, same solid) and only an unchanged `static_mesh.describe` bounds reading kept the bake valid. Ask stands unchanged; the "there is a workaround" mitigation in `#2` should not be read as reducing severity.
