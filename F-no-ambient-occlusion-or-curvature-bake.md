---
id: F-no-ambient-occlusion-or-curvature-bake
title: "No verb anywhere bakes ambient occlusion, cavity, curvature or thickness — not into vertex colours, not into a texture — so a mesh built from interpenetrating .pwmodel parts has no contact shadow at any part junction and no way to author one"
status: OPEN
severity: Medium
category: feature
tags: [geometry, texture, static_mesh, pwmodel, ambient-occlusion, cavity, curvature, thickness, bake, vertex-color, contact-shadow, missing-capability]
encounters: 1
lastSeen: 2026-09-05T19:54:25Z
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
