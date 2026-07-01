---
id: E-get-splines-info-omits-scattered-meshes
title: "spline wiki documents no output schema for get_splines_info / scatter_meshes_along_spline and never routes scatter verify-after-mutate to actor.get_components — so a scatter-then-readback task reaches for the geometry-only verb instead of the component verb that already shows the scattered meshes"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, spline, scatter_meshes_along_spline, get_splines_info, readback, verify-after-mutate, wiki]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# The spline wiki doesn't route scatter confirmation to `actor.get_components`

`spline.scatter_meshes_along_spline` builds `MeshCount+1` separate, **plain**
`UStaticMeshComponent`s and attaches each to the actor's `USplineComponent`
(`SplineHandler.cpp:1265-1283` — `NewObject<UStaticMeshComponent>(Actor)` →
`SetStaticMesh` → `RegisterComponent` → `AddInstanceComponent` →
`AttachToComponent(SplineComp)`), returning `{meshesCreated, splineLength,
spacing}` plus the standard `AddActorVerification` fields (`actorPath` /
`actorGuid` / `existsAfter`) in its own response (`:1287-1294`).

`spline.get_splines_info` reports spline **geometry only**: the named-actor
branch (`:1383-1406`) emits `actorName`, `pointCount`, `splineLength`,
`closedLoop`, and `points[]`; the no-arg list branch (`:1410-1450`) emits
per-actor `splineComponentCount` / `pointCount` / `splineLength` (plus
spline-mesh state for spline-mesh-only actors). It does not enumerate the
attached static-mesh components. That is **correct and honest** — its
registered summary (`:1339`) is "Get information about spline components on
actors in the world", i.e. it is about the `USplineComponent` geometry, not
attached static meshes. There is no false-claim contradiction to fix here.

The actual gap is **discovery/docs, not a missing readback**: a caller who runs
the natural "scatter, then read back to confirm" loop reaches for
`get_splines_info` (the in-namespace read verb) and finds it shows only the
curve — but the scattered meshes are fully readable through a sibling verb. The
scatter products are ordinary scene components on the actor, and
`actor.get_components` already enumerates **every** component on an instance
(`ComponentHandler.cpp:370` `for (UActorComponent* Comp : Found->GetComponents())`),
emitting `name`, `class`, `path`, and — for scene components, which a
`UStaticMeshComponent` is — `relativeLocation` / `relativeRotation` /
`relativeScale` (`:381-403`), plus a `count` (`:409`) and a `componentClass` /
`nameMatch` filter (`:326-329`). So a single
`actor.get_components {actorName, componentClass: "StaticMeshComponent"}` call
returns the full scattered-mesh set with transforms — strictly **more** than the
original draft's proposed `meshComponents[]{name, meshPath, location}` (it adds
rotation + scale + the class filter; only the assigned `UStaticMesh` asset path
is absent, and that is the input the caller themselves passed to the scatter
verb). Combined with the scatter call's own `meshesCreated` + `actorPath`, the
verify-after-mutate workflow is fully served today.

This is why the fix is docs-only, not a `get_splines_info` code change.
Widening `get_splines_info` to enumerate the attached `UStaticMeshComponent`s
would duplicate `actor.get_components` (a dedup / maintenance liability for
negligible payoff). That matches the maintainers' established precedent on the
directly analogous ticket `E-blueprint-get-omits-components-readback-guidance`
(IN-REVIEW) — same "prescribed readback can't see the component I just added; I
fell back to a sibling verb" shape, resolved by routing component readback to
the correct verb + docs, **not** by duplicating component enumeration into the
summary verb. It does **not** match `E-foliage-get-instances-drops-scale`
(IN-REVIEW), which had to widen its readback only because foliage instances are
`FFoliageInstance` HISM entries, not `UActorComponent`s, so **no** sibling verb
enumerates them (that ticket's own workaround is "inspect the
`InstancedFoliageActor` instance buffer directly"). Here a clean sibling verb
exists, so the foliage code-widening precedent does not transfer.

It is distinct from `B-get-splines-info-ignores-spline-mesh` (IN-REVIEW), which
covers `USplineMeshComponent`s created by the **separate**
`create_spline_mesh_actor` verb (its fix is already in the tree at
`SplineHandler.cpp:1367-1377` / `:1433-1445`, adding `isSplineMesh`) — those are
spline-mesh root components, not the plain attached `UStaticMeshComponent`
scatter this ticket is about.

**Workaround:** to confirm a scatter, trust the `scatter_meshes_along_spline`
response's `meshesCreated`, or call `actor.get_components` on the spline actor
and count the `StaticMeshComponent` entries — `get_splines_info` shows the curve
only.

**Docs-only.** The overlay `docs/wiki-src/spline.md` was a bare two-sentence
prelude with no documented workflow or output schema. Add:

- a `## Verifying a scatter` section (renders on the namespace page) routing
  scatter confirmation to (a) the scatter call's own `meshesCreated` and
  (b) `actor.get_components`, and warning that `get_splines_info` shows the
  curve only;
- a `### spline.get_splines_info` section documenting that it returns spline
  geometry only (named-actor and list-branch fields, plus the spline-mesh
  fallback), and stating explicitly that it does **not** enumerate the static
  meshes `scatter_meshes_along_spline` attaches — pointing the reader at
  `actor.get_components` for those; and
- a `### spline.scatter_meshes_along_spline` section documenting its own response
  (`meshesCreated` + verification fields) and a verify-after-mutate note routing
  scatter confirmation to (a) the scatter call's own `meshesCreated`/`actorPath`
  and (b) `actor.get_components {componentClass: "StaticMeshComponent"}` on the
  spline actor, and warning that `get_splines_info` will not show the scattered
  meshes.

No `get_splines_info` code change — `actor.get_components` already returns the
scattered components (name/class/path/location/rotation/scale, with a class
filter), so widening the geometry verb would only duplicate it.

## Evidence (this task — focus `spline.scatter_meshes_along_spline`)

The lightbulb-walkway task (7 calls, outcome clean, friction "none") ran the
exact scatter-then-readback chain the story prescribed:
`create_spline_actor (Curve, 3 pts)` → `add_spline_point (4th)` →
`configure_mesh_spacing` → `configure_mesh_randomization` →
`scatter_meshes_along_spline (SM_Lightbulb, spacing 250 → 9 meshes)` →
`get_splines_info`. Every call returned `ok:true`. The verbatim friction note:

> "get_splines_info reports only spline geometry, not the scattered instance
> count/mesh path, so that confirmation relied solely on the scatter call's
> own response."

The final `get_splines_info` summary was "WalkwayLightString -> 4 Curve pts,
len 2217" — four points and a length, no acknowledgement of the 9 scattered
SM_Lightbulb instances. The task picked the geometry verb to confirm a scatter;
the right confirmation surface is `actor.get_components` (which would have
listed the 9 StaticMeshComponents with transforms) plus the scatter call's own
`meshesCreated`. That routing was undocumented because `spline.md` carried no
output schema or verify-after-mutate guidance at all.

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of the lightbulb-walkway scatter task (focus `spline.scatter_meshes_along_spline`, 7 calls, outcome clean, no judge ticket / `filed_id` empty). PROCESS finding: the task's step (6) asked to confirm BOTH spline points and scattered mesh instances via `spline.get_splines_info`, but source-confirmed (`SplineHandler.cpp`) the named-actor branch (`:1383-1406`) emits only `actorName/pointCount/splineLength/closedLoop/points[]` and never enumerates the `UStaticMeshComponent`s that `scatter_meshes_along_spline` attaches (`:1265-1294`); the no-arg list branch (`:1410-1450`) likewise reports geometry/spline-mesh state only. So the scatter-result confirmation rests solely on the scatter call's own transient `meshesCreated` — the readback the task is pointed at cannot corroborate it. Friction note (verbatim): "get_splines_info reports only spline geometry, not the scattered instance count/mesh path, so that confirmation relied solely on the scatter call's own response." Same scatter-then-verify readback-thinness family as OPEN `E-foliage-get-instances-drops-scale`, but sharper (zero rows about scattered meshes vs. foliage's rows-minus-scale). Dedup (ripgrep over OPEN+closed; qmd unavailable): distinct from `B-get-splines-info-ignores-spline-mesh` (IN-REVIEW — covers `USplineMeshComponent` from the separate `create_spline_mesh_actor` verb, not attached static-mesh scatter), `E-spline-create-actorname-echoes-deduped` (label-collision wrong-target on create), and `E-water-spline-default-points-undocumented` (WaterSpline default-point count). No existing ticket covers scattered-mesh enumeration in `get_splines_info`. Proposed: emit a `scatteredMeshes` count + `meshComponents[]` (`{name, meshPath, location}`) from the readback; document `get_splines_info`'s geometry-only output and the scatter-confirmation caveat in `docs/wiki-src/spline.md` (currently a bare two-sentence overlay with no output schema).
- `#2-reword-docs-only` `IN-REVIEW` developer — Reworded from a `get_splines_info` code-widening to **docs-only** and implemented. Root cause re-scoped: the scattered meshes are plain `UStaticMeshComponent`s (`SplineHandler.cpp:1273` `NewObject<UStaticMeshComponent>(Actor)` + `AddInstanceComponent` `:1279`), and `actor.get_components` already enumerates every component on the actor (`ComponentHandler.cpp:370`) with `name`/`class`/`path` + scene-component `relativeLocation`/`relativeRotation`/`relativeScale` (`:381-403`), a `count` (`:409`), and a `componentClass`/`nameMatch` filter (`:326-329`) — strictly more than the original `{name, meshPath, location}` proposal. `get_splines_info`'s registered summary (`:1339`) is honestly "spline components" geometry, so there is no false-claim to correct (unlike `B-get-splines-info-ignores-spline-mesh`). Widening `get_splines_info` would duplicate `actor.get_components`; the maintainer precedent on the analogous `E-blueprint-get-omits-components-readback-guidance` (#3) routes component readback to the correct sibling verb + docs rather than duplicating enumeration, and the `E-foliage-get-instances-drops-scale` code-widen precedent does not transfer (foliage instances are HISM `FFoliageInstance` entries with no sibling verb; these are real components). FIX (docs): rewrote `Docs/wiki-src/spline.md` (previously a bare 2-sentence prelude) to add a `### spline.get_splines_info` section (documents its geometry-only output + the named/list/spline-mesh branches and states it does not list scattered static meshes, pointing at `actor.get_components`) and a `### spline.scatter_meshes_along_spline` section (documents its own response + a verify-after-mutate note routing scatter confirmation to `meshesCreated`/`actorPath` and `actor.get_components {componentClass:"StaticMeshComponent"}`, warning that `get_splines_info` will not show the scattered meshes). No source/code change and no regression test: a prose wiki-overlay edit has no production code path to exercise. Severity Low / category ergonomic unchanged (pure docs/discoverability friction).
- `#3-reword-docs-only-second-host` `IN-REVIEW` developer — Second independent reword by a parallel fix host, reconciled into this ticket during cross-host rebase (its `## Verifying a scatter` namespace-page section was merged into `spline.md` alongside #2's H3 method sections; neither host's doc was lost). Re-scoped from the proposed `get_splines_info` code-widening to **docs-only** after review, and dropped the code change. Rationale: the scatter IS confirmable today — the `scatter_meshes_along_spline` response carries `meshesCreated`, and `actor.get_components` on the spline actor enumerates every scattered `StaticMeshComponent` with its location/rotation/**scale** (more than the proposed `scatteredMeshes[]{name,meshPath,location}`). `get_splines_info` returns zero scatter rows, so there is no false-confirmation hazard (opposite of the foliage sibling, which silently drops a field while still returning rows). Widening the spline-geometry reader to list attached static meshes would duplicate `actor.get_components` and layer onto the in-flight `B-get-splines-info-ignores-spline-mesh` handler for a Low/ergonomic payoff. The real, proportionate gap is the bare overlay. Implemented: `docs/wiki-src/spline.md` now adds a `## Verifying a scatter` section (renders on the namespace page) plus `### spline.scatter_meshes_along_spline` and `### spline.get_splines_info` H3 sections documenting each verb's output schema and steering scatter-confirmation to `meshesCreated` / `actor.get_components`. Prelude left untouched (2 sentences, within the wiki structural rules). No source change; no regression test (docs-only). Distinct from and complementary to `B-get-splines-info-ignores-spline-mesh` (spline-mesh roots in the fallback branch).
