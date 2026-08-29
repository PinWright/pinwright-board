---
id: B-ground-instances-default-component-foreign-scatter
title: "`spatial.ground_instances` with `component` omitted seats whichever component on the actor carries the most instances — on the level's singleton `AInstancedFoliageActor` that is every caller's foliage, and it relocated 2,048 instances of other agents' authored content across three incidents in one session"
status: OPEN
severity: Critical
category: bug
tags: [spatial, ground_instances, ism, hism, foliage, shared-state, concurrency, multi-agent, silent-mutation, undo, data-loss, default-scope, component-identity]
encounters: 3
lastSeen: 2026-08-29T00:00:00+05:00
---

# The documented default resolves scope by "biggest wins", and on the foliage actor that is never reliably yours

```js
spatial.ground_instances {actorName:"InstancedFoliageActor_0", surface:{preset:"landscape"}}
// no `component` -> 512 instances moved, none of them mine
```

The 512 belonged to another agent's zone roughly **16 km** away, on the same level. They were moved
before the response named the component it had picked.

## Three incidents, one root cause, one session

1. **Zone E — 512 instances moved**, belonging to a zone ~16 km away. **Unrecoverable:** the owning
   agent rewrote that component (1534 -> 170 instances) before it was noticed, so replaying
   `movedInstances[]` by index would have written stale transforms onto their newer data.
2. **Zone A — near miss, same mechanism at the type layer.** `foliage.add_instances` given a mesh
   path reuses a shared `/Game/Foliage/Auto_<Mesh>` type, so `SM_BlueFlower`, `Var24` and `Var6`
   were already living in other agents' components before this agent touched them.
3. **Zone B — 1,536 instances belonging to another agent re-seated**, moved 13-23 cm in Z. That
   agent had inferred component indices from creation order while a second agent was concurrently
   adding types. **Unrecoverable for the same reason:** the owner had re-scattered those components
   (1122 -> 128) by the time it surfaced.

Incidents 1 and 3 are 2,048 instances of other callers' authored content relocated, with no working
recovery in either case. Incidents 1 and 3 were reported by different agents that did not coordinate.

## Mechanism, confirmed in source

`Handlers/Spatial/GroundPlacementHandler.cpp:1265-1269` declares `component` optional; its
description, verbatim at `:1266-1268`:

> Object name of the InstancedStaticMesh/HierarchicalInstancedStaticMesh component to seat
> (case-insensitive). **Omit to use the component carrying the MOST instances** - the same one
> spatial.ground_actors names when it refuses the holder.

Resolution is called at `:1366-1368` into `InstancedMeshUtils::ResolveInstancedComponent`
(`Handlers/Actor/InstancedMeshUtils.h:72-129`). It sweeps
`Actor->GetComponents<UInstancedStaticMeshComponent>` at `:84`, keeps
`if (!Largest || Component->GetInstanceCount() > Largest->GetInstanceCount())` at `:100-104`, and
selects `ComponentName.IsEmpty() ? Largest : Named` at `:127`. The doc comment at `:61-71` states the
rule and, at `:68-69`, the reason it reaches foliage: *"UHierarchicalInstancedStaticMeshComponent and
UFoliageInstancedStaticMeshComponent both derive from UInstancedStaticMeshComponent, so one query
covers every scatter shape."*

**Why that default is unsound specifically on this actor.** `AInstancedFoliageActor` is a per-level
singleton holding one component per foliage type for *every* caller in the level. Because
`UFoliageInstancedStaticMeshComponent` derives from `UInstancedStaticMeshComponent`, the sweep at
`:84` enumerates the whole level's foliage and "largest" picks whichever type happens to be biggest
right now. That is a property of the level, not of the caller. **"Most instances" is never reliably
yours**, and on the foliage actor it is not even stable between two calls, because another agent
painting a different type changes the answer.

## The mesh is the key, so two callers who never coordinate still collide

`foliage.add_instances` given a mesh path auto-creates its foliage type at
`/Game/Foliage/Auto_<MeshBaseName>` — `FoliageHandler.cpp:1189`, and the same construction in
`foliage.paint` at `:372`. The path is derived from the **mesh basename** and from nothing else: no
caller id, no zone, no folder scope. Two agents who never name the same asset, never speak, and work
16 km apart on the level still land in one component the moment they reach for the same mesh — which
in a vegetation library is the normal case, not the unlucky one.

That is the sharpest statement of the hazard: the shared `AInstancedFoliageActor` singleton supplies
the shared *actor*, and the mesh-keyed `Auto_<Mesh>` name supplies the shared *component*. Neither
requires the callers to have done anything wrong. (See `B-add-instances-auto-foliage-type-name-mismatch`,
OPEN/High, which indicts the same construction for a different defect — the package stem and the
object name disagree.)

## There is no supported way to scope, and all three legs were checked

1. **`foliage.add_instances` returns no component name and no index range.** Its complete response
   field set, `Handlers/Environment/FoliageHandler.cpp:1239-1266`: `success` (`:1240`),
   `instances_count` (`:1241`), `skippedCount` (`:1245`), optional `skipped[]` / `skippedTruncated`
   (`:1247-1248`), optional `ignoredLocationCount` / `warnings` (`:1254-1260`), `foliageActorPath`
   (`:1264`), `foliageTypePath` (`:1265`), `existsAfter` (`:1266`). A caller who has just painted
   instances knows the actor and the *type*, and nothing that `ground_instances` accepts.
2. **`AInstancedFoliageActor::FoliageInfos` is `private:` and not a `UPROPERTY`** —
   `C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/InstancedFoliageActor.h:40-43` (`private:` at
   `:40`, the `TMap<TObjectPtr<UFoliageType>, TUniqueObj<FFoliageInfo>>` at `:43`). Unreachable from
   Python and from reflection.
3. **`UFoliageType_InstancedStaticMesh` reflects `ComponentClass`, not the live component** —
   `C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/FoliageType_InstancedStaticMesh.h:28-29`, a
   `TSubclassOf<UFoliageInstancedStaticMeshComponent>`. A class, not an instance.

So the only shape reachable from the foliage verbs is the one with `component` omitted — the exact
shape whose scope is decided by a level-wide popularity contest.

## Component numbering is not stable within a session, so the one available workaround is actively wrong

The single strategy left to a caller is to infer the component from creation order — count the
components, assume the newest suffix is the one just added. **Reported by the zone-B agent, from live
observation rather than from source:** the actor's component set churned from **5 to 57** components
during one session, and the suffix `_7` was `SM_Grass_2` carrying 128 instances at one point and
`Var8` carrying 5,560 instances twenty minutes later.

Nothing in the resolution path contradicts that or could prevent it: `ResolveInstancedComponent`
matches on `Component->GetName()` (`InstancedMeshUtils.h:96`) and holds no identity beyond the string,
so a reassigned suffix **still resolves cleanly**. There is no stale-handle error to hit. The
workaround therefore does not merely lack support — under concurrency it is wrong, and it fails
silently, which is how incident 3 moved 1,536 foreign instances.

## The design decision this ticket contests

`GroundPlacementHandler.cpp:1228-1233`:

> It is deliberately NOT split into a verify/apply pair the way spatial.ground_actors is split from
> spatial.verify_grounding. That split exists because a NAME PATTERN selector on a mutating verb
> needs an expectedMatches guard and a non-mutating sibling to find the number with. Instances are
> addressed by explicit index against one named component on one named actor, **so there is no
> pattern hazard to guard** - and apply:false covers the verify half by running the identical solve
> and reporting the move it would have made.

Restated on the wire in the `apply` param text (`:1273-1279`, the claim at `:1277-1279`) and echoed
approvingly in `F-ism-per-instance-transforms`, which names the resolution as a feature:
*"`GroundPlacement::FindInstancedHolder` … already picks the component carrying the most instances"*
and argues at its `:120-123` that the verify/apply split is unnecessary *"because instances are
addressed by explicit index against one named component, so there is no pattern hazard"*.

Without heat: **the reasoning is correct for an explicit `component`, and the default is the case it
does not cover.** "One named component" is the premise, and the verb's own documented default is that
the component is *not* named — it is inferred, from mutable level-wide state, by a heuristic. The
inference is the pattern hazard, wearing a different name.

## The actor-side precedent, half-received here

`B-ground-actors-prefix-captures-foreign-actors` (**DONE**, High, `encounters: 3`) is this incident
one namespace over. Its `:41-44`:

> There is no "these are the actors I am about to move" surface either: at `detail: "summary"` the
> response carries counts and nothing else, so the six foreign names never appeared. They only
> surfaced on the *next* call, a `verify_grounding` at `detail: "failures"`, by which point the move
> had happened. Had the six seated cleanly they would never have shown up at all.

Its fix (`#3-scope-guard-and-undo-record`, verified `DONE` at `#4`) shipped **two** things:

- **`expectedMatches`** — a pre-flight scope guard that refuses `MISSING_REQUIRED_PARAM` when a
  pattern selector is used without it and `MATCH_COUNT_MISMATCH` *before the first move* when the
  count disagrees, carrying `matchedActors[]`;
- **`movedActors[]`** with `previousTransform` at every detail level.

`spatial.ground_instances` received only the second. The handler comment memorialising the original
incident is still in source, at `GroundPlacementHandler.cpp:569-574`:

> The pre-move transform is echoed whenever one was captured - SUCCESSES INCLUDED. "The move
> succeeded" and "the move was intended" are different facts, and this is the only undo the verb
> has: a batch selected by name prefix once seated six actors belonging to someone else in the same
> level, and because every one of them counted as a success the response carried nothing to put them
> back with. Gating recovery information on failure is what made that unrecoverable.

That comment sits in the shared serializer for all three ground verbs. The lesson it records reached
the receipt half of `ground_instances` and not the guard half. **This is the instance-side sibling,
not a reopening of that DONE ticket.**

## Why the receipt did not save us, and why that is new

The receipt exists and is honest: `movedInstances[]` rows are collected at
`GroundPlacementHandler.cpp:1478-1485` and written at `:1579-1582`, one
`{index, previousTransform:{location,rotation,scale}}` per instance actually moved, ungated by
`detail`. It was still useless in both unrecoverable incidents, for a reason none of the existing
tickets cover.

**The rows are index-keyed, and index identity is not stable across a third-party write.** In
incident 1 the owner had rewritten the component from 1534 instances to 170; in incident 3 from 1122
to 128. Replaying `{index: 903, previousTransform: …}` against a rewritten component writes a stale
transform onto *their* new instance 903. The receipt does not fail loudly in that case — it succeeds,
and corrupts.

**Correction worth recording, because it narrows the ask.** The response is not identity-free: the
handler calls `InstancedMeshUtils::WriteComponentIdentity(Data, Actor, Component)` at `:1551`, which
emits `actor`, `actorPath`, `component`, `componentClass` and `instanceCount` at the top level
(`InstancedMeshUtils.h:199-214`, the three component fields at `:209-212`). So a caller who kept the
**whole response** does hold the component name and its instance count at the moment of the move, and
could detect the 1534 -> 170 rewrite. Three things remain true anyway:

- `WriteComponentIdentity` runs at `:1551`, **after** the move loop closes at `:1541`. The identity
  is a post-mortem, never a pre-flight — which is precisely what `:41-44` above says made the
  actor-side incident unrecoverable.
- The identity is on the envelope, not on the rows. A `movedInstances[]` array lifted out of its
  response is not self-describing, and a same-count rewrite defeats even the envelope check.
- The name it records is not a stable handle, per the churn section above: a component name can be
  reassigned to a different foliage type between the move and the replay, and both resolve.

`B-ism-undo-record-unsafe` (OPEN, High) audits whether these rows are *replayable* — no space marker
(its gap 1), no `space` echo (gap 2), `{index}`-only no-ops (gap 5), the transaction argument. It
does not cover **index invalidation by a third-party rewrite**: a row that is perfectly well-formed,
in the right space, replayed through the right verb, and lands on a different instance than it
recorded. That strengthens its gap 1 rather than duplicating it — gap 1 says the row does not say
*what space* it is in; this says the row does not say *what it was pointing at*.

## Ask, two parts

**(a) `foliage.add_instances` must return the component name and the index range it wrote.** This is
the missing half that makes scoping possible at all: without it there is no legal value to pass for
`component`, so no amount of caller discipline reaches the safe path. The fields exist on the object
already — the handler holds `Info` and could publish `Info->GetComponent()->GetName()` and
`[firstIndex, firstIndex + Added)` beside the `foliageTypePath` it already emits at `:1265`.

**(b) `spatial.ground_instances` must not silently seat a component the caller did not name.** In
descending order of preference:

1. **Require `component`** when the actor carries more than one instanced component. A one-component
   actor has no ambiguity and should keep the convenience.
2. **Apply the actor-side `expectedMatches` guard to the resolved component** — the caller states the
   instance count it expects, and a mismatch refuses `MATCH_COUNT_MISMATCH` *before the first move*,
   naming the component and its true count. This is the shape already shipped and verified one
   namespace over, and it is the only proposal here that also catches the reassigned-name case.
3. **At minimum, move the identity before the mutation** — resolve, then emit the component name,
   class, owning foliage type and instance count, and only then move. `WriteComponentIdentity`
   already produces the block; it is called 10 lines too late.

`F-grounding-holder-not-seatable` uses the same "most instances" heuristic
(`GroundPlacement::FindInstancedHolder`, `Handlers/Spatial/GroundPlacementUtils.cpp:619-655`, the
largest-wins test at `:640-648`) only to **refuse** — to name, in an error message, the component a
caller has to go and fix. **Refusing on a heuristic is safe; seating on one is not.** A wrong guess
in the refusal costs a confusing error string. A wrong guess in the seat costs someone else's
scatter.

## Same shape as

`B-ground-actors-prefix-captures-foreign-actors` (DONE) — same incident, actors instead of instances,
name prefix instead of a largest-wins default. Both are: a mutating verb whose scope is *inferred*
from mutable level-wide state, with the inference reported only after the mutation.

## Cross-links

- `B-ground-actors-prefix-captures-foreign-actors` (DONE, High, `encounters: 3`) — the precedent and
  the shipped fix shape. Do not reopen; this is the instance-side sibling.
- `B-ism-undo-record-unsafe` (OPEN, High) — the receipt audit. The index-invalidation finding above
  is a sixth gap for it; see the encounter appended there.
- `B-add-instances-auto-foliage-type-name-mismatch` (OPEN, High) — same `Auto_<Mesh>` construction
  (`FoliageHandler.cpp:372`, `:1189`), different defect. Whoever changes that naming should read both:
  a caller-scoped or zone-scoped type name would close incident 2's collision *and* that ticket's
  path mismatch in one change.
- `F-ism-per-instance-transforms` (IN-REVIEW, High) — carries the "no pattern hazard" reasoning this
  ticket contests, and approvingly names the largest-wins resolution.
- `F-grounding-holder-not-seatable` — same heuristic used only to refuse.
- `B-foliage-remove-empties-ledger-not-component` (OPEN, Critical) — compounding: after a
  `foliage.remove`, ledger and component indices desynchronise, so the index a `movedInstances[]` row
  records and the index `foliage.get_instances` reports are already different things.

## Severity

**Critical**, argued against the README's band rather than asserted.

*Critical: a write that corrupts or loses asset data.* Across three incidents in one session this
verb's documented default relocated **2,048 instances** of authored content belonging to callers who
did not make the call, and in two of the three the content could not be restored. The instances are
level data and the loss is of the same kind the band exists for. Two further facts make it a property
of the verb rather than of the callers: **no scoping mechanism exists** (all three routes closed —
`add_instances` publishes no component, `FoliageInfos` is private, `ComponentClass` is a class), and
**the one strategy a caller can improvise is actively wrong under concurrency and fails silently**
(component names are reassigned within a session and a reassigned name still resolves).

*The High band — silent false-success / silent wrong data — also fits*, and it is what an earlier
draft of this ticket rated. It is the correct floor if you read the incidents as recoverable.

**The counter-consideration, stated plainly.** In both unrecoverable cases what destroyed the
recovery route was a *third party's concurrent rewrite* invalidating the indices. A single-caller
session would have kept a usable `movedInstances[]` receipt, so the loss is not guaranteed by the
call in isolation.

**Decision: it does not reduce the rating.** The defect is precisely that the verb reaches across
caller boundaries — a shared singleton actor, a mesh-keyed component name, and a scope inferred from
level-wide state. The multi-caller case *is* the case; it is the condition the defect is made of, not
an unlucky aggravation layered on top. Rating it as though callers were isolated would rate a
different bug. Three independent incidents in one session, two unrecoverable, is the evidence that
the isolated reading is the unrealistic one.

**Note on `encounters`.** It is set to 3 to record the three observations, per the schema. It is a
same-severity work-ordering tiebreak and is **not** an input to the rating above; the Critical call
rests on impact and reach alone and would stand at `encounters: 1`.

**Reach modifier declined.** `spatial.ground_instances` is the only per-instance seating verb and the
natural follow-on to `scatter_layout` and `foliage.add_instances`, so it is on a common path — but it
runs in scatter-building sessions, not in almost every session, so no upward bump is earned (and
Critical is the ceiling regardless). It is plainly not a rare edge path, so no downward bump: the
affected shape *is* the documented default and the only one the foliage verbs can produce.

## History
- `#1-default-component-moved-a-foreign-scatter` `OPEN` reporter — Three incidents, one root cause,
  one session. (1) Zone E: `spatial.ground_instances` with `component` omitted, on
  `InstancedFoliageActor_0`, moved 512 instances belonging to a zone ~16 km away; unrecoverable, the
  owner had rewritten the component 1534 -> 170. (2) Zone A: near miss at the type layer — the shared
  mesh-keyed `/Game/Foliage/Auto_<Mesh>` type meant `SM_BlueFlower`, `Var24` and `Var6` were already
  in other agents' components. (3) Zone B: 1,536 foreign instances re-seated 13-23 cm in Z after
  inferring component indices from creation order; unrecoverable, the owner had re-scattered
  1122 -> 128. Mechanism confirmed in source: largest-wins resolution
  (`InstancedMeshUtils.h:100-104`, selected at `:127`) over a level-singleton actor holding every
  caller's foliage; the shared type path at `FoliageHandler.cpp:1189` / `:372` is keyed on the mesh
  basename alone. All three scoping routes confirmed closed (`add_instances` response fields
  `FoliageHandler.cpp:1239-1266`; `FoliageInfos` private and non-`UPROPERTY`,
  `InstancedFoliageActor.h:40-43`; `ComponentClass` is a class not a component,
  `FoliageType_InstancedStaticMesh.h:28-29`). Component-name churn (5 -> 57 components; `_7` was
  `SM_Grass_2`/128 then `Var8`/5,560) is reported from live observation by the zone-B agent, not
  measured here — but `ResolveInstancedComponent` matches on `GetName()` only
  (`InstancedMeshUtils.h:96`), so a reassigned name resolves silently and nothing in source prevents
  it. **Correction recorded against the original report:** the response is not identity-free —
  `WriteComponentIdentity` (`GroundPlacementHandler.cpp:1551`, `InstancedMeshUtils.h:209-212`) emits
  `component` / `componentClass` / `instanceCount` at the top level, but it runs *after* the move loop
  closes at `:1541`, so it is a post-mortem and not a pre-flight, the identity is on the envelope
  rather than on the `movedInstances[]` rows, and the name it records is not a stable handle. New
  finding for `B-ism-undo-record-unsafe`: index-keyed rows are invalidated by a third party's
  rewrite, and replaying them then succeeds while corrupting the newer data. Rated **Critical** on the
  data-loss band; the counter-argument (a solo caller would have had a working receipt) is recorded
  and rejected, because cross-caller reach is the defect itself rather than an aggravating accident.
