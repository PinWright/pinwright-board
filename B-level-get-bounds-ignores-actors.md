---
id: B-level-get-bounds-ignores-actors
title: "level.get_bounds returns a degenerate all-zero box on any level without a LevelBounds actor — never sums actor bounds despite documenting 'encloses every actor'"
status: IN-REVIEW
severity: High
category: bug
tags: [level, bounds, silent-wrong-result, contract-violation]
---

# level.get_bounds returns an all-zero box when there is no LevelBounds actor

`level.get_bounds` documents (wiki + handler description verbatim): *"Return the
world-space axis-aligned bounding box that **encloses every actor in the level**
(origin + extent)."* The implementation does not do that. It only reads the
level's `ALevelBounds` actor and, when that actor does not exist, returns a
zero-initialized `FBox` — i.e. `min=(0,0,0) max=(0,0,0)`.

Every freshly created level (`level.create`) has **no** `ALevelBounds` actor by
default, so `get_bounds` silently reports a degenerate origin-anchored zero box
for the entire normal case, regardless of how many actors the level contains or
where they are. The call returns `ok` (no error), so a caller has no signal that
the answer is wrong — it is a silent success-with-a-bogus-result that
contradicts the method's own documented contract.

This is **not** the "lights have no primitive bounds" edge case. The handler
never iterates the level's actors at all, so even an actor with real renderable
geometry placed far from the origin does not move the reported bounds off
`(0,0,0)..(0,0,0)`.

## Handler source

`Source/PinWright/Private/Handlers/Level/LevelHandler.cpp`,
`level.get_bounds` (around line 1382):

```cpp
FBox LevelBounds(ForceInit);                 // zero-init: min=max=(0,0,0)
if (TargetLevel->LevelBoundsActor.IsValid())
{
    LevelBounds = TargetLevel->LevelBoundsActor->GetComponentsBoundingBox();
}
// ... reports LevelBounds.Min / LevelBounds.Max with no actor-iteration fallback
```

There is no fallback path that sums `Actor->GetComponentsBoundingBox()` over
`TargetLevel->Actors`, so a level with no `ALevelBounds` actor always reports the
zero box.

## Verbatim repro (replayed via mcp__editor-automation__call)

1. `level.create` `{"levelName":"GetBoundsReplay"}`
   -> `{"levelPath":"/Game/Maps/GetBoundsReplay", ...}`
2. `level.get_info` `{}` -> `{"actorCount":11}` (level already has 11 actors)
3. `level.get_bounds` `{}` ->
   `{"min":"X=0.000000 Y=0.000000 Z=0.000000","max":"X=0.000000 Y=0.000000 Z=0.000000"}`
   (degenerate zero box despite 11 actors present)
4. `actor.spawn` `{"classPath":"/Script/Engine.StaticMeshActor","meshPath":"/Engine/BasicShapes/Cube.Cube","location":{"x":1000,"y":2000,"z":300}}`
   -> spawns a Cube with real geometry at (1000,2000,300)
5. `level.get_info` `{}` -> `{"actorCount":12}`
6. `level.get_bounds` `{}` ->
   `{"min":"X=0.000000 Y=0.000000 Z=0.000000","max":"X=0.000000 Y=0.000000 Z=0.000000"}`
   — STILL all zeros even with a non-origin mesh actor present. The handler does
   not enclose the actors at all.

## What it should do

When `TargetLevel->LevelBoundsActor` is invalid/absent, fall back to summing
every actor's `GetComponentsBoundingBox()` over `TargetLevel->Actors` (the
behavior the documentation already promises, and what `ALevelBounds` itself does
internally via its auto-update). Alternatively, if the intent is strictly to
report the `LevelBounds` actor's box, the doc must say so and the result must
flag the missing-actor case (e.g. a `hasLevelBounds:false` field) instead of
returning a silent zero AABB that reads as a valid measurement.

**Workaround:** spawn an `ALevelBounds` actor, but note its auto-calc also
ignores actors that contribute no level-bounds-relevant primitive bounds (e.g.
lights), so callers end up manually disabling `bAutoUpdateBounds` and sizing the
box by hand — a multi-call detour to get a number the method claims to return.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed against mcp__editor-automation__call: fresh `level.create` level reports `level.get_bounds` min=max=(0,0,0) with 11 actors present; still all zeros after spawning a StaticMeshActor Cube at (1000,2000,300) [actorCount 12]. Handler `LevelHandler.cpp` `level.get_bounds` zero-inits `FBox` and only reads `TargetLevel->LevelBoundsActor`, with no actor-iteration fallback — contradicts the documented "encloses every actor in the level".
- `#2-actor-iteration-fallback` `IN-REVIEW` developer — Added an actor-iteration fallback to `level.get_bounds` in `Source/EditorAutomationRpcGateway/Private/Handlers/Level/LevelHandler.cpp`: when `TargetLevel->LevelBoundsActor` is absent or yields an invalid box, sum each actor's `GetComponentsBoundingBox()` over `TargetLevel->Actors` (seed `FBox(ForceInit)`, skip `ALevelScriptActor`, union only `ActorBounds.IsValid` boxes) — mirrors the auto-calculate path in `LevelStructureHandler.cpp:698-714`. Added honest `hasLevelBounds` and `isValid` result fields and updated the handler description; added `#include "GameFramework/LevelScriptActor.h"`. Regression test `EditorAutomationRpcGateway.level.get_bounds.EnclosesActors` (FLevelGetBoundsEnclosesActorsTest) in `Source/EditorAutomationRpcGateway/Private/Tests/World/TestLevelHandlers.cpp` spawns a cube StaticMeshActor at (4000,5000,600) into the live editor world, runs the production handler, and asserts the returned box is valid and `IsInsideOrOn` the actor location — fails against the reverted LevelBoundsActor-only handler (returns the degenerate origin box that excludes the actor).
- `#3-additional-lights-excluded` `IN-REVIEW` reporter — Additional evidence (the lights sub-case is still live after the #2 fallback shipped, and is now the remaining defect): the fallback's shared `McpActorUtils::SumActorBounds` (`Source/.../Private/Utils/ActorUtils.cpp` `AccumulateActorBounds`, line ~25) calls `Actor->GetComponentsBoundingBox()` with the **default** `bNonColliding=false`, which drops lights' non-colliding editor components — so a level whose only non-builder actors are lights still reports just the builder-brush box, breaking the "encloses every actor" contract for that case. This is *inconsistent with the sibling* `actor.get_bounding_box`, which calls `GetActorBounds(false, ...)` (i.e. `bOnlyCollidingComponents=false`, *includes* non-colliding) in `Private/Handlers/Actor/ActorTransformHandler.cpp:145` and is documented as "including all primitive components (visible and hidden)". Replayed live against mcp__editor-automation__call on level `/Game/Maps/CourtyardNight` (3 lights: a PointLight at 0,0,400 and two SpotLights at ±~746): `level.get_bounds` `{}` -> `isValid:true, hasLevelBounds:false, min=(-128,-128,-128) max=(128,128,128)` (only the 256-cube builder brush). Yet `actor.get_bounding_box` `{"actorName":"CourtyardCenterPoint"}` -> `origin [0,0,400] extent [128,128,128]` (reaches z=528, outside 128) and `{"actorName":"CourtyardAccentSpotA"}` -> `origin [-746.43,-746.43,294.30] extent [181.57,181.57,133.70]` (reaches ~-928,-928). So three actors with finite renderable bounds via the sibling RPC are entirely excluded from `level.get_bounds`. Fix: pass `bNonColliding=true` to `GetComponentsBoundingBox()` in `AccumulateActorBounds` (both `SumActorBounds` overloads) so it matches `actor.get_bounding_box`'s `GetActorBounds(false,...)` and the documented contract.
- `#4-additional-doc-says-origin-extent-result-is-min-max` `IN-REVIEW` reporter — Additional evidence (ergonomic doc/output mismatch, distinct from the box-value defects above and to fix alongside the #2 rename if convenient): the #2 actor-iteration fallback now works — replayed live on `/Game/Maps/Material/Material_Nodes` (Content Examples demo map, 151 actors), `level.get_bounds` `{}` -> `{"levelPath":"/Game/Maps/Material/Material_Nodes","hasLevelBounds":false,"isValid":true,"min":"X=-773.692612 Y=-1121.478400 Z=-825.638828","max":"X=18050.000000 Y=6000.000000 Z=6194.000000"}` (correct non-zero summed box). But the wiki page (`level.get_bounds.md`) and the handler `REGISTER_RPC_HANDLER` summary string (`LevelHandler.cpp:1405`) both promise the box verbatim as **"(origin + extent)"**, while the result object carries no `origin`/`extent` fields at all — only `min`/`max`, each a *string* `"X=%f Y=%f Z=%f"` (`LevelHandler.cpp:1460-1461`), not a numeric vector. A caller following the doc to "record the returned origin and extent" finds neither field and must (a) string-parse the `X=/Y=/Z=` corners and (b) derive `origin=(min+max)/2`, `extent=(max-min)/2` by hand. The sibling `actor.get_bounding_box` already reports numeric `origin`/`extent` arrays, so the two bounds RPCs disagree on both field names and encoding. Fix when touching this handler: either emit numeric `origin`/`extent` (matching the doc and the sibling) — keeping `min`/`max` as numeric arrays too if useful — or change the doc to say "min/max corners (string-encoded)". As-is the doc text is concretely misleading about the result shape.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
