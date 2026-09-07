---
id: E-duplicate-along-spline-undiscoverable
title: "geometry.duplicate_along_spline is undiscoverable — 'distribute copies along the path' is searched in spline.*/actor.*, and the geometry.array_linear/array_radial steer-note points only at manual actor.duplicate+set_transform, never cross-linking the purpose-built verb"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, geometry, spline, duplicate_along_spline, discoverability, cross-namespace-discovery, wiki]
encounters: 1
costly: 1
lastSeen: 2026-07-02T03:26:06.4106887+03:00
---

# `geometry.duplicate_along_spline` is the one-call op for "populate a spline path with aligned, scale-varied actor copies" — but it lives in `geometry.*` and nothing routes a caller there

`geometry.duplicate_along_spline` (params: `actorName`, `splineActorName`,
`count`, `alignToSpline` default true, `scaleVariation`) is the purpose-built,
single-call operation for exactly the "lay out matching copies along a curve,
facing along it, with slight size variation" intent. In the audited park-bollard
task it was never found — the agent emulated the entire operation in ~18 RPC
calls plus hand-rolled quaternion→yaw and scale-variation math, because the
method is undiscoverable from where a caller naturally looks.

## Why it's undiscoverable (the discovery paths all miss it)

1. **A caller scopes discovery to `spline.*` and `actor.*`.** The audited agent's
   own narration (SAY, transcript line 67): *"Let me focus on the relevant
   namespaces: spline (curve path + scattering) and actor"* — the obvious home
   for "distribute along a path." But in `spline.*` the only along-spline verb is
   `spline.scatter_meshes_along_spline`, which attaches mesh **components** (not
   separate placeable actors — its own wiki note says `get_splines_info` "does not
   enumerate" them, so it fails an `actor.find_by_name` success-check), and in
   `actor.*` `actor.duplicate` has no spline mode. Neither namespace points at
   `geometry.duplicate_along_spline`.

2. **The `geometry` page the agent DID read steers to the manual pattern with NO
   cross-link.** `Docs/wiki-src/geometry.md` `### geometry.array_linear` (line 58)
   and `### geometry.array_radial` (line 64) each end with the verbatim steer:
   *"To get N separate, individually placeable actors instead, use
   `actor.duplicate` + `actor.set_transform` per copy."* — pointing straight at
   the manual per-copy fallback the agent then hand-rolled, and never mentioning
   `geometry.duplicate_along_spline` for the spline case. `Docs/wiki-src/spline.md`
   (which has a `## Verifying a scatter` section and H3s for scatter /
   get_splines_info) likewise carries **no** cross-link to
   `geometry.duplicate_along_spline` for "separate placeable actors along a
   spline."

Net: a 1-call operation was emulated in ~18 RPC calls (`actor.get_components` +
10 `object.call_function GetTransformAtDistanceAlongSpline` samples + 8
`actor.duplicate` + 8 `actor.set_transform`) plus a source dive into engine
`SplineComponent.h` for the `GetTransformAtDistanceAlongSpline` UFUNCTION
signature and coord-space enum, plus manual `yaw = 2*atan2(Z,W)` and per-post
scale math. The agent's friction note (verbatim): *"there is no
...array-actors-along-spline helper, so I evaluated the spline through
object.call_function."* The helper exists; it just could not be found.

## Applicability caveat (must be documented alongside the cross-link)

Even once found, `geometry.duplicate_along_spline` requires a **DynamicMeshActor**
source. The natural template here — `actor.spawn` of `/Engine/BasicShapes/Cylinder`
— yields a **StaticMeshActor**, which the verb would reject. So the discoverability
fix only pays off if the reader also knows to author the template via
`geometry.create_cylinder` (which produces a DynamicMeshActor). A cross-link that
omits this constraint would send a caller to a verb that rejects their StaticMesh
template. (Separately, the verb's `scaleVariation` branch clobbers the source's
base scale — filed as `B-duplicate-along-spline-scalevariation-clobbers-base-scale`.)

## What it should be (docs-only; the fix is a downstream wiki edit)

- **`Docs/wiki-src/spline.md`** — add a short "distributing separate placeable
  actors along a spline" note cross-linking `geometry.duplicate_along_spline`
  (contrast it with `scatter_meshes_along_spline`, which makes non-enumerable mesh
  components, not actors).
- **`Docs/wiki-src/geometry.md`** — extend the `array_linear` / `array_radial`
  steer-notes (lines 58 / 64): for the **spline** case, point at
  `geometry.duplicate_along_spline` (one call: count + alignToSpline +
  scaleVariation) rather than only the manual `actor.duplicate` +
  `actor.set_transform` per-copy pattern; keep the manual pattern for the
  non-spline case.
- **`geometry.duplicate_along_spline` page** — state the source must be a
  **DynamicMeshActor** (a StaticMeshActor spawned from a mesh path won't qualify;
  author the template via `geometry.create_cylinder`/etc.).

severity rationale: impact=discoverability/docs (a working, purpose-built verb the
caller can't find; a documented manual workaround exists and the agent used it
successfully — not a blocker) × reach=common level-dressing intent but not an
every-session method → Low. (Symptom cost was real — ~18 calls + a source dive to
emulate a 1-call op — which is why it is worth fixing even at Low.)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a park-bollard-along-a-spline
  task (focus `geometry.duplicate_along_spline`; 42 calls; outcome done via the
  hand-rolled emulation). PROCESS finding (from the CallAnalyzer + friction note):
  the FOCUS/purpose-built verb was never used despite being the exact one-call op
  for the goal — a pure cross-namespace discoverability miss (it lives in
  `geometry.*`, not the `spline.*`/`actor.*` the agent searched), compounded by the
  `geometry.md` array_linear/array_radial steer-note (lines 58/64) pointing only at
  the manual `actor.duplicate`+`actor.set_transform` fallback with no cross-link to
  `geometry.duplicate_along_spline`, and `spline.md` carrying no cross-link either.
  Confirmed live in the wiki source. Result: ~18 RPC calls + a `SplineComponent.h`
  source dive + manual quaternion→yaw/scale math to emulate one call. Applicability
  caveat: the verb needs a DynamicMeshActor source, so the natural
  `/Engine/BasicShapes/Cylinder` StaticMeshActor template wouldn't have qualified
  without switching to `geometry.create_cylinder` — the cross-link must say so.
  Dedup: ripgrep over OPEN+closed found `B-duplicate-along-spline-scalevariation-clobbers-base-scale`
  (the scaleVariation-overwrites-scale BUG — different: a wrong-data defect, not
  discoverability), `E-get-splines-info-omits-scattered-meshes` (scatter readback
  routing, IN-REVIEW — different verb/intent), and `E-geometry-array-radial-merges-in-place`
  (which authored the very steer-note this ticket wants extended, IN-REVIEW — that
  ticket's scope is the in-place-merge semantic mismatch, not the missing
  duplicate_along_spline cross-link). No existing ticket covers
  `geometry.duplicate_along_spline` discoverability. Proposed: cross-link it from
  `spline.md` and the `geometry.md` array steer-notes, and document the
  DynamicMeshActor-source constraint.
