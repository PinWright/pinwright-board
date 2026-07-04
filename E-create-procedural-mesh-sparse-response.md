---
id: E-create-procedural-mesh-sparse-response
title: "geometry.create_procedural_mesh (+7 sibling create verbs) skip AddActorVerification — response omits actorPath/actorGuid"
status: OPEN
severity: Low
category: ergonomic
tags: [geometry, create_procedural_mesh, revolve, response-shape, actorpath, actorguid, AddActorVerification, consistency]
---

# Half the `geometry.create_*` family omits `AddActorVerification`, so their responses carry no actor object path or GUID

`geometry.create_procedural_mesh` returns only `{name, class, enableCollision}`
(PrimitiveHandler.cpp:818-822) — the spawned `DynamicMeshActor`'s object path,
GUID, and map path are absent. The sibling spawn verbs `actor.spawn` /
`actor.spawn_shape` return the rich `AddActorVerification` set
(`actorPath`/`mapPath`/`actorName`/`actorGuid`/`existsAfter`/`actorClass`,
AssetUtils.cpp:997-1018). A caller that needs the object path or GUID (to
reference the actor in a level, or to disambiguate a colliding label) can't get
it from this response and must issue a follow-up query.

This is an **intra-file inconsistency**, not just a cross-namespace gap: 8 of
the ~16 geometry create verbs already call `AddActorVerification(Result, NewActor)`
before `SendSuccess` — `create_box` (:149), `create_sphere` (:188),
`create_cylinder` (:229), `create_cone` (:271), `create_capsule` (:320),
`create_plane` (:401), `create_disc` (:437), `create_stairs` (:479) — while 8
omit it: `create_torus` (:328-362), `create_spiral_stairs`, `create_ring`,
`create_arch`, `create_pipe`, `create_ramp`, `revolve` (:779-784), and
`create_procedural_mesh` (:818-822). Same handler pattern, same available
`NewActor`, inconsistent result shape.

No hard block observed — the mesh stays addressable by its label for subsequent
`geometry.*` calls — so this is a consistency/discoverability gap, not a
blocker. But a build that needs the path/guid pays a round-trip on 8 of the
create verbs and not the other 8.

**Workaround:** address the mesh by name for follow-up `geometry.*` calls, or
query `actor.find_by_*` for the object path/guid when needed.
**Fix:** add `AddActorVerification(Result, NewActor);` before `Ctx.SendSuccess`
in each of the 8 omitting create verbs, matching the 8 that already call it.

## History
- `#1-sparse-response-repro` `OPEN` reporter — Observed `call("geometry.create_procedural_mesh", {name:"SmokeTetra", location:{x:-600,y:0,z:100}})` return `{"name":"SmokeTetra","class":"DynamicMeshActor","enableCollision":false}` — no `actorPath`, no `actorGuid`, and the `location` arg not echoed. Subsequent calls addressed the mesh by name successfully (no hard block). Source verified: PrimitiveHandler.cpp:818-822 (`create_procedural_mesh`) and :779-784 (`revolve`) build their result without `AddActorVerification`, unlike `create_box` (:149) / `create_sphere` (:188) and 6 other create verbs that call it; the helper (AssetUtils.cpp:997-1018) would supply `actorPath`/`mapPath`/`actorGuid`/`existsAfter`/`actorClass`. Full omitter set: create_torus, create_spiral_stairs, create_ring, create_arch, create_pipe, create_ramp, revolve, create_procedural_mesh (8 verbs). Distinct from `E-geometry-create-name-vs-actorname` (input param `name` vs `actorName`), `E-geometry-deformer-echo-mesh-counts` (mutators omit vertex/tri counts), and `E-actor-verification-actorpath-is-map-path` (fixes the helper's `actorPath` value). Fix: reuse `AddActorVerification(Result, NewActor)` in the 8 omitting verbs.
