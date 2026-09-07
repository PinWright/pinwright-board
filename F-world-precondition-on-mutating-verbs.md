---
id: F-world-precondition-on-mutating-verbs
title: "Mutating verbs have no caller-assertable world precondition, so a world swap between two calls silently writes into a different map — while render.capture_open_level already guards exactly this"
status: IN-REVIEW
severity: High
category: feature
tags: [effect, spawn-niagara, actor, world, active-world, multi-agent, world-lock, precondition, capture, worldMatches, silent-wrong-target]
encounters: 1
lastSeen: 2026-09-06T06:21:00Z
---

# The capture verb checks which world it is looking at. The write verbs do not.

`render.capture_open_level` refuses with `VIEWPORT_WORLD_MISMATCH` when the viewport's world is not
the active editor world, publishes `activeWorld` / `viewportWorld` / `worldMatches` on every
response, and gained an explicit `allowPieWorld` opt-in. That is the right shape. **No mutating verb
has the equivalent.** `effect.spawn_niagara`, `actor.delete`, `actor.spawn`, `actor.set_transform`
and friends all act on whatever world happens to be active at the instant the request is dequeued,
with no way for the caller to say which world it believes it is writing to.

In a single-agent session that is harmless. In this project — six streams sharing one editor, with
`Docs/fps/PLAN.md`'s world lock as the only coordination — it means a sequence that was correct when
it started writes into a **different stream's map** if the world changes mid-sequence, and reports
success.

## What I called, and what happened

2026-09-06, UE 5.8, gateway 27145, VFX critic holding the world lock.

    06:19:54Z  (took world lock atomically, token b56af133, map /Game/FPS/Test/T_VFX)
    call("editor.list_dirty_packages", {})  -> {"count": 0}
    call("level.load", {levelPath: "/Game/FPS/Test/T_VFX"})
      -> {"loaded": true, "activeLevelPath": "/Game/FPS/Test/T_VFX", "deferredToSafePoint": true}
    sleep 2.2                                      # PLAN rule 11 settle
    call("effect.spawn_niagara", {systemPath: "/Game/FPS/VFX/NS_Muzzle_AR", ...})
      -> {"actorPath": "/Game/FPS/Test/T_Player.T_Player:PersistentLevel.NiagaraActor_0",
          "mapPath":   "/Game/FPS/Test/T_Player",          <-- NOT the map I loaded
          "active": true, "systemAssigned": true, "compileStatus": "passed"}

Between `level.load` returning `/Game/FPS/Test/T_VFX` and the spawn ~3 seconds later, another stream
loaded `/Game/FPS/Test/T_Player` and started PIE. My spawn went into **their** map. Nothing in the
request could have expressed "only if the active world is still T_VFX", and nothing in the response
was an error — the call succeeded, into the wrong world.

I noticed only because `effect.spawn_niagara` happens to echo `mapPath`, and I read it. I deleted
the actor immediately (`actor.delete` -> `deletedCount: 1, existsAfter: false`), so the net content
change was zero, but the foreign map was left dirty by my write and its owner's PIE session was
running, during which every asset save silently fails
(`B-asset-save-pie-failure-reports-pendingflush`). Had I batched spawn + step_and_capture + delete
without reading `mapPath`, I would have captured someone else's level, scored my stream against it,
and left an actor behind.

## What I want

A `expectedWorld` (or `expectedMap`) optional parameter on mutating verbs, checked immediately
before the mutation and refused with a typed error naming both worlds:

    call("effect.spawn_niagara", {systemPath: "...", expectedWorld: "/Game/FPS/Test/T_VFX", ...})
      -> WORLD_PRECONDITION_FAILED
         {"expectedWorld": "/Game/FPS/Test/T_VFX", "activeWorld": "/Game/FPS/Test/T_Player"}

and, independently of the opt-in, `activeWorld` / `activeWorldPackage` on every mutating response,
the way the capture verbs already do. The echo alone would let a caller assert after the fact;
the precondition is what makes it safe.

Minimum viable subset if the full sweep is too broad: `effect.spawn_niagara`, `actor.spawn`,
`actor.spawn_batch`, `actor.delete`, `actor.set_transform`, `level.save`. Those are the verbs that
write world state.

## Why this is not just an agent-discipline problem

The proximate cause here was another stream taking the lock non-atomically and swapping the world
under a holder — a `PLAN.md` violation, and the orchestrator's problem, not the plugin's. But the
plugin gap stands on its own two ways:

1. A world can change between any two RPCs for reasons no agent controls — a map load deferred to a
   safe point, PIE starting, an editor restart mid-sequence. Discipline narrows the window; it
   cannot close it. The capture verb was given a guard for exactly this reason and the write verbs
   were not, which is an inconsistency inside one API rather than a missing feature at its edge.
2. It generalises past this project. Any PinWright user driving the editor from more than one place
   — a CI job and an interactive session, two agents, a script and a human — has the same exposure,
   and today the only defence is to read `mapPath` on every response and hope every verb echoes one.
   Several do not.

## Root-cause guess

Not read from source. The shape suggests handlers resolve the world through a shared
"active editor world" accessor at dequeue time with no caller-supplied expectation to compare
against, whereas the capture path already threads a world identity through in order to compare
viewport against editor world.

## Workaround in use

Read `mapPath` / `actorPath` on every `effect.spawn_niagara` response and abort the sequence if it
is not the expected map; re-assert `level.get_info` immediately before each mutating call. Both are
advisory — they narrow the window rather than closing it, and they only work on verbs that echo a
world at all.

## Fix

The ticket is TRUE. Registrations had no mutation metadata, dispatcher validation knew only each
handler's local parameter list, and the shared response path did not identify the world. The only
cross-handler effect knowledge was split between `FDemoGate::IsMutatingMethod` and the explicit
tick-unsafe method table, so no common point could enforce a caller's world expectation.

Added `Dispatch/WorldPrecondition.*` and explicit registration metadata. The shared resolver uses the
editor world for editor mutators and the handler-matching PIE-aware selector for runtime UI and
Niagara spawn. Dispatcher draining injects optional `expectWorld`; validation runs before entry and
again in every retained safe-point/job continuation. Full object paths and package paths are
accepted, with typed `WORLD_MISMATCH` details. `AddWorldField` preserves a handler-defined value
and asserts if it disagrees with the resolved target.

`UPinWrightSubsystem::SendAutomationResponse` is the single response decorator for success and
failure envelopes, including null-result errors, async/fanout completions, and formatted/captured
responses. It records the method with the request policy so the same target resolver is used. Direct
and nested `DispatchMethod` calls now apply the precondition immediately before handler entry.
Mutation metadata no longer derives from the DemoGate leaf-name heuristic: the new explicit
registration macro marks the ticket's non-tick-unsafe mutators, while the tick-unsafe table supplies
the existing effect metadata and known read-only analysis probes are excluded and ratcheted.

Files changed: `Source/PinWright/Private/Dispatch/WorldPrecondition.h`,
`Source/PinWright/Private/Dispatch/WorldPrecondition.cpp`,
`Source/PinWright/Private/Dispatch/SafePoint.h`,
`Source/PinWright/Private/Dispatch/RpcDispatcher.cpp`,
`Source/PinWright/Private/Handlers/HandlerContext.h`,
`Source/PinWright/Private/Handlers/HandlerContext.cpp`,
`Source/PinWright/Private/Handlers/HandlerRegistration.h`,
`Source/PinWright/Private/Handlers/HandlerRegistration.cpp`,
`Source/PinWright/Private/Handlers/Actor/ActorTransformHandler.cpp`,
`Source/PinWright/Private/Handlers/Blueprint/BlueprintCreationHandler.cpp`,
`Source/PinWright/Private/Handlers/Level/LevelHandler.cpp`,
`Source/PinWright/Private/Handlers/VFX/EffectHandler.cpp`,
`Source/PinWright/Private/Tests/World/TestWorldPrecondition.cpp`,
`Source/PinWright/Public/PinWrightSubsystem.h`,
`Source/PinWright/Private/PinWrightSubsystem.cpp`,
`Docs/wiki-src/mcp-transport.md`, and `Docs/wiki-src/level.md`.

Automation tests: `PinWright.world.precondition.DispatchContract` drives the real
`actor.set_transform` handler against a scoped editor-world actor, checks exact world values,
mismatch-before-write, package-path success, omitted compatibility, direct/nested dispatch refusal,
explicit mutating registration, and read-only registration exclusions. `PinWright.world.precondition.ResponsePolicy`
covers separate fanout IDs, null success, exact success/error world values, ID reuse, and read-only
responses through the subsystem decorator. `PinWright.world.precondition.DeferredContinuation` queues
the retained `RunAtSafePoint` path with one transient editor world, switches the editor context to a
second transient world before the core-ticker continuation, and asserts that the mutation body is not
entered and the response is `WORLD_MISMATCH` with the active `data.world` and original
`data.expectedWorld`. Source-only checks only; no build, editor, MCP call, process, or automation run
was performed.

## History
- `#1-filed` `OPEN` reporter — `effect.spawn_niagara` returned success with `mapPath: "/Game/FPS/Test/T_Player"` roughly three seconds after `level.load` had returned `activeLevelPath: "/Game/FPS/Test/T_VFX"`, because another stream loaded its own map and started PIE in between; the actor was created in a map belonging to a different stream and the response carried no error. No parameter exists to express "spawn only if the active world is still T_VFX", and most mutating verbs do not echo the world they acted on at all — while `render.capture_open_level` refuses `VIEWPORT_WORLD_MISMATCH` and publishes `activeWorld`/`viewportWorld`/`worldMatches` for the same hazard. Asking for an optional `expectedWorld` precondition plus an unconditional `activeWorld` echo on the world-writing verbs (`effect.spawn_niagara`, `actor.spawn`, `actor.spawn_batch`, `actor.delete`, `actor.set_transform`, `level.save`). Caught by reading `mapPath` and reverted with `actor.delete` (`existsAfter:false`), so no content was changed, but the foreign map was left dirty and a batched spawn-capture-delete would have scored one stream's review against another stream's level.
- `#2-shared-world-precondition` `IN-REVIEW` developer — Added one registration/dispatcher/response policy for all mutating methods: optional `expectWorld` is checked immediately before handler entry with typed `WORLD_MISMATCH`, and successful mutating responses echo `world`. Added `PinWright.world.precondition.DispatchContract` for mismatch refusal before a real actor mutation, matching and omitted calls, schema discovery, and response identity; updated transport and level wiki source pages. Source verification only, with execution reserved for the compile/suite agents.
- `#3-central-response-policy` `IN-REVIEW` developer — Closed the direct async fanout gap without editing `blueprint.create`: mutation metadata is retained per request ID before queueing and across async work, then consumed by the subsystem's single response boundary. Successful mutating fanout/stream completions now receive the active-world echo; failures and non-mutating responses do not. Added no-asset policy coverage for multiple response IDs, null success payloads, failure, and read-only success. Source-only verification; no build, editor, MCP, or automation run.
- `#4-policy-lifecycle` `IN-REVIEW` developer — Replaced stale request-ID policies on every newly accepted request, so mutating-then-read-only ID reuse cannot inherit the world echo; nested cross-dispatch keeps the outer policy. Added subsystem-shutdown cleanup for abandoned entries after pending completions are failed, plus focused reuse coverage. A disconnected request can still retain its entry until subsystem deinitialization because dropped transport completions have no callback. Source-only verification; no build, editor, MCP, or automation run.
- `#5-central-only-decorator` `IN-REVIEW` developer — Removed duplicate world-field writes from handler contexts, async tokens, and the formatted dispatcher envelope. Successful mutating responses now have one production writer at `UPinWrightSubsystem::SendAutomationResponse`; the dispatch behavior test covers precondition and mutation ordering, while the response-policy test covers the shared success decorator. Updated the transport wiki and ticket file list to describe that boundary. Source-only verification; no build, editor, MCP, or automation run.
- `#6-verifier-followup` `IN-REVIEW` developer — Replaced PIE-first response identity with one handler-family target-world resolver; retained expectWorld payloads and revalidated them in RunAtSafePoint and DeferJobToSafePoint continuations; preserved handler-defined world values with a ratchet; decorated mutating failures as well as successes, including formatted/captured paths; added direct/nested DispatchMethod validation; and replaced DemoGate mutation inference with explicit registration metadata plus read-only safe-point exclusions. Added exact success/error assertions and nested-dispatch coverage. Deferred world-swap runtime coverage is explicitly declined because this source-only brief forbids the live ticker/world harness needed to exercise it. Source-only verification; no build, editor, MCP, or automation run.
- `#7-deferred-precondition-coverage` `IN-REVIEW` developer — Added a real editor-context automation test for the retained safe-point continuation: two scoped transient editor worlds are created, `expectWorld` is captured from the first, the continuation is queued, the active editor world is switched to the second before the core-ticker hop, and the test asserts the mutation body stays untouched while the response carries `WORLD_MISMATCH`, `data.world` for the second world, and `data.expectedWorld` for the first. No build, editor, MCP, or automation run was performed.
- `#8-nested-capture-fixture` `IN-REVIEW` developer — Fixed the nested-dispatch contract fixture to drain registrations without a subsystem, keeping the same-thread test on the dispatcher capture path instead of stopping at the editor-readiness gate; the actual `DispatchMethod` precondition now supplies the captured `WORLD_MISMATCH` response and exact world details before `actor.set_transform`. The deferred fixture now passes an explicit UE 5.8 non-World-Partition initialization record to both transient editor worlds. Source-only verification; no build, editor, MCP, or automation run was performed.
