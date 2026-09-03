---
id: F-render-runtime-spawned-actor
title: "No verb renders an actor that is not placed in the active level — nothing captures an actor class or a runtime-spawned actor without dirtying the user's map"
status: OPEN
severity: Medium
category: feature
tags: [render, capture, preview, actor, blueprint, runtime-spawn, visual-verification, active-world]
encounters: 2
lastSeen: 2026-08-18T00:00:00Z
---

# The capture surface covers assets and the active level, and nothing in between

Current capture verbs:

- `render.capture_asset_preview` — **Static Mesh asset editors only**. Anything else is rejected
  with a typed error naming `render.capture_animation_preview`
  (`Handlers/Render/RenderHandler.cpp:212`, rejection at `:277-280`).
- `render.capture_animation_preview` — Persona / skeletal preview.
- `render.capture_open_level`, `editor.screenshot`, `render.capture_annotated` — whatever is
  already sitting in the active level viewport.
- `asset.generate_thumbnail` (`Handlers/Asset/AssetWorkflowHandler.cpp:578`) — the nearest
  existing capability, and it does render an actor Blueprint offscreen. It is a *thumbnail*: no
  camera placement, no framing to bounds, no exact-size output — and it renders the **asset**, not
  an actor instance carrying the state it only has once spawned.

Nothing answers "show me this actor". An actor that exists only at runtime — spawned by a game
mode, a spawner, or a data payload — cannot be rendered at all.

## Why it came up, and why it is worth a verb

This session's football world (arena, ball, both goals, six spawn points, 28 boost pickups) was
spawned at runtime from a JSON payload and placed in no map, so no capture verb could see any of
it while the visual work — ball scale, boost-pickup size and emissive — was exactly what needed
looking at.

**Two agents independently invented the same workaround**: spawn a stand-in (`DFH_BoxPreview`)
into the active level, `render.capture_open_level`, then delete it. Independent reinvention of the
same three-step dance is the signal here — the shape is obvious enough that everyone derives it,
which is the argument for it living in the plugin rather than in every caller.

The workaround is also unsafe in the way that matters: it **mutates the user's active level to
take a read-only picture**, dirtying the map and, if any step fails, leaving the stand-in behind.
This is the same "read path that mutates the active world" objection
`E-level-getters-require-loaded-not-on-disk` argued from, here paid in a dirty map. It is
particularly bad against a live editor — this session's editor was serving the user's own flight
testing.

**Requested:** `render.capture_actor_preview` taking either `classPath` (spawn a transient
instance) or `actorPath` (an existing actor, including one in a PIE world), framing the camera to
the actor's bounds the way `B-capture-asset-preview-renders-empty`'s fix already frames a static
mesh, capturing at an exact size, and destroying/restoring on **every** exit path. It should reuse
`PreviewViewportCaptureUtils` — realtime override, pump loop, `ReadPixels`, `imageStats`, blank
classification — rather than growing a second capture path with its own bugs.

## Related

- `B-capture-asset-preview-renders-empty` (IN-REVIEW) — its Workaround section already names this
  gap from the other direction: *"for mesh visual review, fall back to spawning the mesh into a
  level and using a level-viewport capture ... which is exactly the no-spawn workflow this verb
  was meant to avoid."* That ticket fixes the static-mesh path; it does not add an actor path.
- `B-inspect-misses-pie-world` (DONE) — closed the same blind spot for `system.inspect.*` and
  `actor.*` queries. The render surface never got the equivalent, so a PIE-world actor can be
  queried but not seen.
- `F-widget-designer-screenshot` (DONE) — precedent shape: a capture verb for a thing with no
  level presence.
- `E-generate-thumbnail-undocumented` (DONE) — the thumbnail verb this is repeatedly mistaken for.

## History
- `#1-two-agents-same-workaround` `OPEN` reporter — No capture verb reaches an actor that is not already in the active level. `render.capture_asset_preview` is Static Mesh asset editors only and rejects everything else with a typed error (`RenderHandler.cpp:212`, `:277-280`); `render.capture_animation_preview` is Persona; `render.capture_open_level` / `editor.screenshot` / `render.capture_annotated` capture whatever is already in the level viewport; `asset.generate_thumbnail` (`AssetWorkflowHandler.cpp:578`) renders the **asset** at thumbnail scale with no camera, framing or exact-size control, not a spawned instance. So an actor that exists only at runtime is unrenderable. Hit this session on the football world (arena, ball, 2 goals, 6 spawn points, 28 boost pickups) — spawned at runtime from a JSON payload, in no map, while ball scale and boost-pickup size/emissive were exactly the things under review. **Two agents independently invented the same `DFH_BoxPreview` spawn-render-delete workaround**, which is the signal: the shape is obvious, so it belongs in the plugin. It also mutates the user's active level to take a read-only picture (dirties the map, leaks the stand-in if any step fails) — the same objection `E-level-getters-require-loaded-not-on-disk` raised against mutating reads, and worse against a live editor, which this session's was. Requested: `render.capture_actor_preview {classPath | actorPath}` — spawn transient or target an existing actor including one in a PIE world, frame to bounds as `B-capture-asset-preview-renders-empty`'s fix already does for static meshes, exact-size capture, destroy/restore on every exit path, reusing `PreviewViewportCaptureUtils` (realtime override, pump, `ReadPixels`, `imageStats`, blank classification) rather than a second capture path. Cross-refs: `B-capture-asset-preview-renders-empty` (IN-REVIEW) already names this gap in its own Workaround section; `B-inspect-misses-pie-world` (DONE) closed the equivalent blind spot for `actor.*`/`system.inspect.*` queries but not for rendering; `F-widget-designer-screenshot` (DONE) is the precedent shape.
