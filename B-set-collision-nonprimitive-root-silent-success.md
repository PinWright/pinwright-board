---
id: B-set-collision-nonprimitive-root-silent-success
title: "`actor.set_collision` falls through both of its guards silently — an actor with no root component, or with a non-primitive root such as the `DefaultSceneRoot` every wiki-recipe HISM holder carries, gets `success` with the REQUESTED `collisionEnabled` echoed back and nothing written"
status: DONE
severity: Medium
category: bug
tags: [actor, set_collision, collision, root-component, rootless, default-scene-root, silent-noop, silent-false-success, missing-echo, request-echo, ism, hism, holder-actor]
encounters: 1
lastSeen: 2026-08-29T21:55:00+03:00
---

# Two guards, no else, and a response that echoes the request

`actor.set_collision` (`Private/Handlers/Actor/ActorPropertyHandler.cpp:182`) writes inside two
nested `if`s and neither has an `else`:

    ActorPropertyHandler.cpp:207   if (USceneComponent* RootComp = Actor->GetRootComponent()) {
    ActorPropertyHandler.cpp:208     if (UPrimitiveComponent* PrimComp = Cast<UPrimitiveComponent>(RootComp)) {
    ActorPropertyHandler.cpp:210       PrimComp->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
    ActorPropertyHandler.cpp:212       PrimComp->SetCollisionEnabled(ECollisionEnabled::NoCollision);

and then, unconditionally:

    ActorPropertyHandler.cpp:217   TSharedPtr<FJsonObject> Data = MakeShared<FJsonObject>();
    ActorPropertyHandler.cpp:218   Data->SetStringField(TEXT("actorName"), ActorName);
    ActorPropertyHandler.cpp:219   Data->SetBoolField(TEXT("collisionEnabled"), bCollisionEnabled);
    ActorPropertyHandler.cpp:220   Ctx.SendSuccess(Data);

`bCollisionEnabled` is the value read off the payload at `:192-193`. It is never re-read from the
component. So the response is an echo of the request, and it is byte-identical whether the write
landed or was skipped entirely. There is no `componentName`, no `rootComponentClass`, no
`applied` list, and no warning.

## Which actors fail

- **No root component at all.** `actor.spawn` of a bare `AActor` produces one on some paths —
  `B-actor-spawn-rootless-actor-ignores-location` documents the same guard shape swallowing
  `location` on the same actor class.
- **A non-primitive root**, which is the common case. `USceneComponent` is not a
  `UPrimitiveComponent`, so any actor whose root is a plain scene component or a `DefaultSceneRoot`
  takes the silent path. That includes **every holder actor built by the plugin's own scatter
  recipe** (`Saved/PinWright/wiki/level-building.instancing-and-scatter.md`): a spawned `AActor`
  with its instanced mesh components attached under a scene root. On the level this was found from,
  `ZoneF_Veg` carries 15 HISM components and 5888 instances under exactly that shape.

For those actors the verb is inert by construction and says nothing.

## Not RPC-verified — source-only

The verb was **not called** in the pass this was found in, and that is the finding's own context:
the task needed per-channel collision on a *named component*, which this verb cannot express
(`F-component-collision-channel-write`), so the pass never reached it. What is asserted here is a
read of `ActorPropertyHandler.cpp:189-221` at HEAD `962275fa`, where the control flow is plain. A
live repro is one `actor.spawn {classPath:"/Script/Engine.Actor"}` followed by
`actor.set_collision {actorName, collisionEnabled:false}` and a `property.get` of
`RootComponent` — the second call will report success on an actor with no primitive to write to.

## Fix

Distinguish the three outcomes instead of collapsing them into one success:

- no root component -> `NO_COMPONENT` (the error code `actor.apply_force` already uses for the same
  condition, `ActorPropertyHandler.cpp:118`), or a success carrying `applied: false` plus a warning
  naming the root's class;
- non-primitive root -> the same, naming `rootComponentClass` so the caller can see *why*;
- success -> echo the value **read back** from `PrimComp->GetCollisionEnabled()`, not the value
  requested, and name the component that was written.

The read-back echo is the part that closes it: `rpc-design.md`'s "report only what happened" and
"verification the write path cannot fake". An echo of the request cannot fail.

## Not a duplicate of

- **`B-actor-spawn-rootless-actor-ignores-location`** (OPEN, Medium) — **the sibling, same failure
  shape, same actor class, different verb and different property.** That one is `location` going
  nowhere on a rootless spawn with no transform in the reply; this one is `collisionEnabled` going
  nowhere with a request-echo in the reply. A fixer could land either without the other; whoever
  takes one should read the other, because the remedy (name the root, echo what was read) is the
  same shape.
- **`F-component-collision-channel-write`** (OPEN, Medium, filed from the same pass) — that no verb
  can address a named component's collision at all. This ticket is about the verb that exists being
  silently inert on a class of actors; that one is about it being the wrong instrument even when it
  works. Split deliberately: merging a silent-false-success bug into a missing-capability request
  would have the picker work the bug at the feature's severity.
- **`B-declared-param-guard-blind-spots`** (IN-REVIEW, High) and
  **`B-verbs-read-undeclared-parameters`** (IN-REVIEW, High) — both name `actor.set_collision`, for
  reading `collision_enabled` / `actor_name` off the raw payload without declaring them. Schema
  hygiene; neither touches the write path or the response.
- **`B-shape-extent-stale-physics`** (IN-REVIEW, High) — documents `actor.set_collision {false}` then
  `{true}` as a workaround for forcing a physics-state rebuild. Worth flagging to whoever fixes
  this: that workaround is **also** silently inert on a non-primitive root, so a caller following
  that ticket's advice on a holder actor gets two successes and no rebuild.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* carries the enumeration. This is a
plain member: the call succeeds, and the deciding fact — that nothing was written — is not merely
unreported but structurally unreportable, because the only field in the response is a copy of the
input.

## Severity

**Medium**, matching `B-actor-spawn-rootless-actor-ignores-location`, which is the same defect on
the same actor class through a different verb.

**Impact = High** on the rubric (silent false-success: the caller trusts a result that is a lie).
**Reach bumps it down one.** `actor.set_collision` is a single-purpose verb, and the failing input
class is a subset of its callers — actors whose root is not a primitive. That is the rubric's "rare
edge path" clause applied honestly rather than generously: it is not exotic (it is every holder
actor the wiki teaches you to build, and every Blueprint actor with a `DefaultSceneRoot`), but it is
not the majority of what this verb is pointed at either. Rating it High would put it ahead of
measured silent-false-success defects on every-session verbs, which is the mis-ordering the rubric's
reach modifier exists to prevent.

**Not Low.** Low is docs, discoverability, naming or a response spill. Nothing here is cosmetic: the
caller believes collision was changed and it was not.

## History
- `#1-two-silent-guards-and-a-request-echo` `OPEN` reporter — Source-only, found while filing
  `F-component-collision-channel-write` from a collision-and-seating fix pass on
  `/Game/Maps/PW_VegetationTest` (plugin source HEAD `962275fa`). The verb was never called in that
  pass — it cannot address a named component, which is why the pass fell back to `python.execute` —
  so this is a read of `ActorPropertyHandler.cpp:189-221`, not an observation, and the body says so.
  Control flow verified line by line: both guards are bare `if`s with no `else`, `bCollisionEnabled`
  comes from `Ctx.GetBoolFirstOf` at `:192-193` and is written straight back to the response at
  `:219` without being re-read from the component. Dedup: searched the board for
  `actor.set_collision`, `set_collision`, `root-component`/`RootComponent`, `rootless`,
  `DefaultSceneRoot`, `silent-noop`, `silent success`, and every `B-actor-*` file. Three tickets
  name the verb — `B-declared-param-guard-blind-spots` and `B-verbs-read-undeclared-parameters` for
  undeclared parameter reads, `B-shape-extent-stale-physics` for using it as a physics-rebuild
  workaround — and none of them concerns what it writes or what it reports.
  `B-actor-spawn-rootless-actor-ignores-location` is the same guard shape on `actor.spawn` and is
  cross-linked rather than appended to, because it is a different verb and a different property and
  either could be fixed alone.
- `#2-refuse-nonprimitive-root-echo-readback` `IN-REVIEW` developer — Premise confirmed at PinWright `7230b41d` (`ActorPropertyHandler.cpp:207-220`, unchanged from the report). `actor.set_collision` now refuses `NO_COMPONENT` when the actor has no root or its root is not a `UPrimitiveComponent`; the error carries `{actorName, rootComponentClass}` (empty when rootless) and the message names the root, says nothing was written, and points at `actor.set_component_properties {componentName, properties:{CollisionEnabled}}`. On success the reply adds `componentName` and `collisionEnabled` is read back via `PrimComp->GetCollisionEnabled()`, so it reads `false` when actor-level collision (`bActorEnableCollision`) overrides an enable. Raw `"NO_COMPONENT"` literal kept to match `actor.apply_force` in the same file; citing `ErrorCodes::ERR_NO_COMPONENT` would flip the file to registry-adopting and fail `PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly` on its other hand-spelled codes. Behaviour change: callers that hit holder actors (and the `B-shape-extent-stale-physics` false/true rebuild workaround on such actors) now get an error instead of a false success. Registration description and `CHANGELOG.md` updated. Files: `Source/PinWright/Private/Handlers/Actor/ActorPropertyHandler.cpp`, new `Source/PinWright/Private/Tests/Actor/TestActorSetCollisionRoot.cpp`, `CHANGELOG.md`. Tests (dispatcher-routed, fail on revert): `PinWright.actor.set_collision.RefusesNonPrimitiveRoot` (DefaultSceneRoot + HISM child, child collision untouched), `PinWright.actor.set_collision.RefusesRootlessActor`, `PinWright.actor.set_collision.EchoesReadBackCollision` (StaticMeshActor root; enable with actor collision off reports `false`). Filter `PinWright.actor.set_collision`.
- `#3-review-fixes-remedy-and-actor-collision-warning` `IN-REVIEW` developer — Review round 1 addressed. The test now reads the component's own setting with `BodyInstance.GetCollisionEnabled(/*bCheckOwner=*/false)` (the owner-checked default read NoCollision under `SetActorEnableCollision(false)` and failed the fixed handler). The refusal's remedy was a non-working call; it is now `actor.set_component_properties {componentName, properties:{BodyInstance:{CollisionEnabled:"NoCollision"}}}`, because `CollisionEnabled` is not a UPROPERTY on the component. The refusal data adds `primitiveComponents[]`, and the rootless message now gives a next step (`actor.add_component`). On success the reply adds `actorEnableCollision`, plus a `warnings[]` entry naming `bActorEnableCollision` when the read-back differs from the request. `EchoesReadBackCollision` and `RefusesNonPrimitiveRoot`/`RefusesRootlessActor` assert all of these. CHANGELOG bullet updated to match.
- `#4-review-r2-nits` `IN-REVIEW` developer — Round-2 nits. The remedy text and the CHANGELOG now spell `{actorName, componentName, properties:{BodyInstance:{CollisionEnabled:...}}}`. A disable while actor-level collision is off is an engine no-op on the component's own setting (`SetCollisionEnabled` compares against the owner-checked NoCollision); it now adds a `warnings` entry saying that setting was left unchanged and returns when actor collision is re-enabled, and `EchoesReadBackCollision` asserts it. The registration summary now mentions `primitiveComponents`.
- `#5-run1-fix-disable-warning-removed` `IN-REVIEW` developer — Run 1: `EchoesReadBackCollision` failed on the round-3 disable-with-actor-collision-off assertions. UE 5.8 applies the actor override inside `FBodyInstance::GetCollisionEnabled_CheckOwner` only through `OwnerComponent` (`BodyInstance.cpp:927-944`). That field is set only by `InitBody` (`:1515`), so on a bodiless component (the mesh-less fixture) `SetCollisionEnabled` writes the disable instead of no-oping. That warning and its assertions are removed. The enable warning is kept and now reports the measured own setting. Summary and CHANGELOG reworded to match.
- `#6-verified-linux` `DONE` tester — Fix commit `b405a9e2`. PinWright `ae877ccc` (on origin/master, base `7230b41d`), UE 5.8 Linux Vulkan. run3/full is the offscreen full suite: 5827/5827 ok, 0 fail, 73 skips, none of them this ticket's tests, and no PINWRIGHT_ASSERTIONS_SKIPPED marker for them. The build (clean unity, 0 errors) and Python (462 OK) come from run2 on the same tree. Passed non-skipped in run3/full: `PinWright.actor.set_collision.RefusesNonPrimitiveRoot`, `.RefusesRootlessActor`, `.EchoesReadBackCollision`, plus the existing `.MissingActorName` and `.ValidParams`. `EchoesReadBackCollision` was run1's only failure; it was fixed under `#5` and passes here. All three Fix outcomes are met. (1) A rootless actor is refused with `NO_COMPONENT`, and `rootComponentClass` is empty. (2) A non-primitive root (a `DefaultSceneRoot` with an HISM child) is refused with `NO_COMPONENT`, naming `rootComponentClass` and listing `primitiveComponents[]`, and the child's collision is untouched. (3) On success the reply names `componentName`, and `collisionEnabled` is read back with `GetCollisionEnabled()`, not echoed. An enable that reads back false while actor collision is off carries `actorEnableCollision:false` and a `warnings[]` entry. All three tests go through the dispatcher and fail on revert. Coverage limits: the refusal's suggested remedy, `actor.set_component_properties {properties:{BodyInstance:{CollisionEnabled}}}`, is not called from these tests. That write path passes separately in `PinWright.actor.set_component_properties.InstanceBodies*`. The `#4` disable-with-actor-collision-off warning was removed in `#5`, because on a bodiless component the engine writes that disable instead of skipping it.
