---
id: E-geometry-array-radial-merges-in-place
title: "geometry.array_radial / array_linear bake copies INTO the source mesh in-place (no actors spawned), but their wiki name + 'copies including original' count doc read as 'spawn N actors' — disambiguate the docs"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [geometry, array_radial, array_linear, dynamic-mesh, in-place-merge, wiki, docs, copies, misleading, semantic-mismatch]
---

# `geometry.array_radial` / `array_linear` merge N copies into the single source mesh's geometry, yet their wiki name + "copies including original" `count` doc strongly imply they spawn N separate actors/copies — a name/doc-vs-behavior semantic mismatch

`geometry.array_radial` does **not** create separate actors (or separate dynamic
meshes) arranged in a ring. It bakes `count` rotated copies of the source mesh
**into the source mesh's own geometry, in place** — the actor count is unchanged
and the source mesh's vertex/triangle count multiplies by `count`. This is a
legitimate UE-Modeling-Mode "pattern/array" operation (one editable merged
mesh), so it is not a bug. The ergonomic trap is that **nothing in the method
name, the wiki, or the response signals "in-place merge" vs. "N instances"** —
and the words that *are* present ("array", "copies including original",
`count:6`) all point the wrong way.

A caller asked to "make a radial set of 6 copies … forming a circular
colonnade, then confirm the new arrayed actors exist" reasonably expects 6
placeable arch actors. Instead they get 1 actor whose mesh is now 6× heavier,
and a success response that gives no hint of this — so the "confirm new actors
exist" step fails with no error to explain why, and the caller burns extra
`actor.list` / `get_mesh_info` probes to discover the mesh silently grew.

## Why this is misleading (quotable demonstration)

Wiki (`Saved/EditorAutomation/wiki/geometry.array_radial.md`), verbatim:
- Description: **"Create a radial array of a dynamic mesh around a center point"**
- `count` (`integer`, optional): **"Number of copies including original (default 6)"**

"copies including original" is the language of *instancing* (N things, the
original plus N-1 more) — it does not say "merged into the source mesh." The
sibling `geometry.array_linear` carries the **identical** wording ("Number of
copies including original (default 3)"), so the same trap applies to the linear
form by documentation.

Success response, verbatim (replayed live):
```
call geometry.array_radial {"actorName":"TempleArch","count":6,"axis":"Z","angle":360}
-> {"actorName":"TempleArch","count":6,"angle":360,"message":"Radial array applied"}
```
The response **echoes the input `count:6`** and says "Radial array applied" —
reinforcing "6 copies now exist." It reports no `vertexCount`, no new actor
names/paths, and no flag like `mergedInPlace` to disclose what actually
happened.

Observed effect (replayed live, same session):
- Before: `actor.list filter=TempleArch` → 1 actor; `get_mesh_info` → 14256 verts / 28368 tris.
- Call `array_radial count=6 axis=Z angle=360` → `"Radial array applied"`, `count:6`.
- After: `actor.list filter=TempleArch` → **still 1 actor** (no `DynamicMeshActor` index above the pre-existing max); `get_mesh_info` → **85536 verts / 170208 tris** (exactly 14256×6 / 28368×6 — six copies baked into the one mesh).

So the operation succeeded and did real, correct work; it is simply
indistinguishable, from name+wiki+response, from the "spawn N actors"
interpretation the wording invites.

## What it should do — Fix (scope: docs/name only)

**Disambiguate the wiki + `count` doc.** Change the `array_radial` /
`array_linear` registered summary and the `count` parameter description to state
the result is a *single merged dynamic mesh* (copies are appended into the source
mesh geometry, no new actors), e.g. "Append `count-1` rotated copies into the
source mesh's own geometry around a center point (one merged dynamic mesh; does
NOT spawn actors)." and `count` → "Total copies including original; count-1
copies are merged into this actor's mesh in place, no new actors are spawned."
These are the auto-generated `RPC_PARAM_OPT` / handler-summary strings the wiki
renders verbatim, so editing them at the source fixes the discovery surface.

**Scope note (rescoped from the original filing):** the *count-echo* half of this
ticket — echoing post-op `vertexCount`/`triangleCount` on the array responses —
has been **folded into `E-geometry-deformer-echo-mesh-counts`**, which is the
family-wide "every geometry mutator should echo its post-op counts (matching
`subdivide`/`simplify_mesh`)" ticket; `array_radial`/`array_linear` are simply two
more members of that same gap and should be solved there, once, not twice. This
ticket now covers ONLY the genuinely-distinct part the deformer-echo ticket does
not: the **name/doc-vs-behavior semantic mismatch** (the verb name "array" + the
"copies including original" `count` doc invite an "N instances/actors"
interpretation, with nothing signalling the in-place merge).

**Workaround / sibling that does the "N separate actors" intent:** there is no
actor-spawning radial-array verb in `geometry`, but the `actor` namespace can
produce the ring of placeable instances: `actor.duplicate` ("Duplicate a placed
actor in the same world … translate the copy by an optional offset") N-1 times,
then `actor.set_transform` to rotate each clone about Z (or
`geometry.duplicate_along_spline` against a circular spline). So the "circular
colonnade of separate arches" task is achievable — just not through
`geometry.array_radial`, whose name and docs are what point an agent there.

## Distinct from related tickets

- `E-geometry-deformer-echo-mesh-counts` (the family-wide *missing count echo*
  ticket) now **owns the count-echo half** of the original filing — the array
  verbs should echo `vertexCount`/`triangleCount` there alongside the deformers.
  This ticket retains only the **name/doc-vs-behavior semantic mismatch**
  (instance-array wording vs. in-place merge), which the deformer-echo ticket
  does not cover — fixing the count echo would not touch the "I expected actors"
  trap.
- `E-actor-duplicate-locked-level-opaque-error` is about `actor.duplicate`'s
  error text on locked levels — unrelated.

## History
- `#1-initial-repro` `OPEN` reporter — Seed task `geometry.create_arch` (ruined-temple "TempleArch" colonnade); attempt outcome failed on step 4 ("make 6 radial copies forming a colonnade, confirm the new arrayed actors exist"). Replay-confirmed live via `mcp__editor-automation__call`: `geometry.array_radial {actorName:TempleArch, count:6, axis:Z, angle:360}` returned `{"actorName":"TempleArch","count":6,"angle":360,"message":"Radial array applied"}` while `actor.list filter=TempleArch` stayed at 1 actor and `geometry.get_mesh_info` jumped 14256v/28368t → 85536v/170208t (exactly ×6 — six copies merged into the one source mesh, zero actors spawned). Classified ergonomic (not bug): the op is a correct UE-Modeling array/pattern merge; the trap is that the wiki ("Create a radial array of a dynamic mesh"; `count`="Number of copies including original") and the `count:6` echo all read as "spawn N copies/actors", with no `vertexCount`/`mergedInPlace`/actor-list signal to the contrary. `geometry.array_linear` shares the identical "copies including original" wording. Workaround for the actual "ring of separate actors" intent: `actor.duplicate` + `actor.set_transform` (rotate about Z) or `geometry.duplicate_along_spline`. Fix: disambiguate the wiki/`count` doc to say "single merged mesh, no actors spawned" and echo post-op counts / a `mergedInPlace` flag.
- `#2-reword-rescope-and-doc-fix` `IN-REVIEW` developer — REWORDED then implemented. Adversarial review flagged the original filing as mis-scoped: the *count-echo* half (echo `vertexCount`/`triangleCount` on the array responses) is a near-duplicate of the family-wide `E-geometry-deformer-echo-mesh-counts` ("every geometry mutator should echo post-op counts like `subdivide`/`simplify_mesh`") — `array_radial`/`array_linear` are just two more members of that same gap. Rescoped this ticket to ONLY the genuinely-distinct part that ticket does not cover: the name/doc-vs-behavior semantic mismatch (the "array" verb name + the auto-generated `count` doc "Number of copies including original" both read as "spawn N instances/actors", with nothing signalling the in-place merge). Source re-verified: `array_radial` appends `count-1` rotated copies INTO `Target.Mesh` via `AppendMeshTransformed` (GeometryTransformHandler.cpp:303-304) + `NotifyMeshUpdated` (:306), zero `SpawnActor`; `array_linear` does the same via `AppendMeshRepeated` (:210-211). FIX (docs/name only, additive): rewrote both registered handler summaries to "Append count-1 ... copies into the source mesh's own geometry ... (one merged dynamic mesh; does NOT spawn actors). To get N separate placeable actors, use actor.duplicate + actor.set_transform instead." and both `count` `RPC_PARAM_OPT` descriptions to "Total copies including original (default N); count-1 ... copies are merged into this actor's mesh in place, no new actors are spawned" — these are the verbatim strings the wiki renders. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Geometry/GeometryTransformHandler.cpp` (:151,:154 array_linear; :225,:228 array_radial). REGRESSION TEST: `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestGeometryArrayDocDisambiguation.cpp` (`EditorAutomationRpcGateway.infra.geometry.ArrayVerbDocDisclosesInPlaceMerge`) — reads the live `FAutoRegisterHandler::GetPendingRegistrations()` records (same data the wiki renders) and asserts both verbs' summary mentions merge/into-source-mesh AND states no actors spawned, and the `count` doc discloses the in-place merge; reverting the description strings to the old "copies including original" wording fails it. Not compiled/tested here (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Private/Handlers/Geometry/GeometryTransformHandler.cpp` → `Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryTransformHandler.cpp`; `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestGeometryArrayDocDisambiguation.cpp` → `Source/PinWrightGeometry/Private/Tests/Geometry/TestGeometryArrayDocDisambiguation.cpp`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
