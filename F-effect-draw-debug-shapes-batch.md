---
id: F-effect-draw-debug-shapes-batch
title: "effect.draw_debug_shape is one-shape-per-call — marking out a scene with N preview shapes costs N round-trips"
status: OPEN
severity: Low
category: feature
tags: [effect, draw_debug_shape, debug-shape, batch, preview, blockout, step-count, docs]
encounters: 1
lastSeen: 2026-06-23T10:10:14Z
---

# `effect.draw_debug_shape` draws exactly one shape per call — a multi-marker blockout is N separate calls

`effect.draw_debug_shape` (`Handlers/VFX/EffectHandler.cpp:283`) takes a single
`shapeType` (`EffectHandler.cpp:286`, read at `:362-363`) and emits one shape per
invocation. There is no array/batch form. So the canonical "block out a scene
with several transient preview markers" intent — which is exactly what these
helpers exist for (`docs/wiki-src/effect.md:3`: *"Runtime/editor preview helpers
… debug shapes … to visualize it before I commit to real geometry"*) — costs one
RPC per shape, even when the shapes are near-identical and differ only by
position.

This is the debug-shape sibling of the batch precedents already on the board
(`F-console-batch-get-cvar-values`, `F-batch-pin-defaults`,
`F-geometry-lod-settings-batch`, `F-vehicle-wheel-asset-batch-properties`): a
clean-outcome, pure step-count gap where every call succeeds but the *intent* was
one logical action ("mark out the pit + flame + three seating spots").

## What's awkward

A single blockout layout fans out into N `draw_debug_shape` calls. In the
evidence task, "mark out the scene with debug shapes" was **5 separate calls**
(circle + arrow + 3 spheres), and the 3 sphere markers were *identical in every
parameter except `location`* (300u out at 0/120/240°) — the textbook case a
batch form collapses. The companion verbs are already whole-scene-scoped
(`effect.clear_debug_shapes` clears *all* shapes in one call;
`effect.list_debug_shapes` lists *all*), so the draw side is the odd one out:
you tear down the whole preview set atomically but must build it up one shape at
a time.

## What it should do

Add a batch draw that takes an array of shape specs and emits them in one round
trip, returning a per-shape result list:

```
effect.draw_debug_shapes(shapes: [ { shapeType, location, rotation?, scale?,
                                     color?, duration?, size?, thickness? }, ... ])
  -> { drawn: [ { index, shapeType, ok, error? }, ... ], count }
```

The handler already has the per-shape dispatch (`EffectHandler.cpp:379-515`); a
batch entry point is a thin loop over the existing single-shape path reusing the
same param parsing, so the marginal cost is small. Keep the singular
`effect.draw_debug_shape` for the one-off case. (A lighter alternative that
covers the common "same shape, many points" case specifically: let the singular
verb accept `location` as an array of points, drawing one shape of the given
`shapeType` per point — but the general array-of-specs form above also handles
the mixed circle+arrow+sphere layout in this task, which a points-only variant
would not.)

## Docs angle (`docs/wiki-src/effect.md`)

`docs/wiki-src/effect.md` is a 13-line namespace prelude (lines 1-13) with **no
per-method sections at all** — nothing documents `draw_debug_shape`,
`clear_debug_shapes`, or `list_debug_shapes`, their params, or the
one-shape-per-call shape. The task only succeeded first-try because the wiki
pages *on disk* (not this overlay) happened to document the params (friction
note: *"wiki pages on disk documented every param (shapeType list, color
[r,g,b,a], duration, …)"*). Until a batch verb lands, an `### effect.draw_debug_shape`
section on `effect.md` should at minimum state it is one shape per call and that
a multi-marker blockout is N calls (with `clear_debug_shapes` as the
whole-set teardown), so the step-count is a documented expectation. The wiki edit
itself is the downstream wiki process, not this ticket.

## Evidence

From a nighttime-campfire blockout struggle audit (namespace `effect`; story:
"mark out the scene with debug shapes … a circle … an arrow … and three sphere
markers in a triangle"; outcome `clean`; friction note verbatim: *"none - wiki
pages on disk documented every param … all calls succeeded first try with no
retries or python.execute fallback."*). 16 calls total; the
shape-drawing phase was **5 consecutive `effect.draw_debug_shape` calls**:

1. `effect.draw_debug_shape {shapeType:circle, origin, r150, orange, dur3600}`
2. `effect.draw_debug_shape {shapeType:arrow, origin, pitch90, orange, dur3600}`
3. `effect.draw_debug_shape {shapeType:sphere, (300,0,0)}`
4. `effect.draw_debug_shape {shapeType:sphere, (-150,259.8,0)}`
5. `effect.draw_debug_shape {shapeType:sphere, (-150,-259.8,0)}`

Calls 3-5 are one logical "three seating-log markers" action expressed as three
RPCs differing only by `location`. Nothing failed — this is pure step-count,
exactly the friction class `F-console-batch-get-cvar-values` and the other batch
tickets capture. A single `effect.draw_debug_shapes([...5 specs...])` would have
made the whole blockout phase one call.

## Not a duplicate of

- `E-effect-actor-name-slot-vs-actorname` (OPEN) — that is the `name` vs
  `actorName` param-drift on `effect.spawn_niagara` / `actor.*` readback; this is
  the missing batch *capability* on `draw_debug_shape`, a different verb and a
  different friction class.
- `F-console-batch-get-cvar-values` / `F-batch-pin-defaults` /
  `F-geometry-lod-settings-batch` — same batch *family*, different namespaces and
  verbs; none touch the `effect.*` debug-shape draw surface.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the nighttime-campfire
  blockout task (namespace `effect`; outcome `clean`; the per-finding judge filed
  nothing — `filed_id` empty — so no OUTCOME overlap). Distinct PROCESS angle:
  pure step-count from the one-shape-per-call `effect.draw_debug_shape`
  (`EffectHandler.cpp:283/286/362`). "Mark out the scene with debug shapes"
  fanned out into 5 consecutive draw calls (circle + arrow + 3 spheres), the 3
  spheres identical except `location` (300u at 0/120/240°). Companion verbs
  `clear_debug_shapes`/`list_debug_shapes` are already whole-scene-scoped (one
  call clears/lists all), so the draw side is the inconsistent one. Propose
  `effect.draw_debug_shapes(shapes:[...])` as a thin loop over the existing
  per-shape dispatch (`:379-515`), keeping the singular verb. Docs fallback:
  `docs/wiki-src/effect.md` is a 13-line prelude with no per-method section for
  any debug-shape verb — add an `### effect.draw_debug_shape` note stating
  one-shape-per-call until the batch verb ships. Same friction class as
  `F-console-batch-get-cvar-values` (clean-outcome step-count batch gap).
