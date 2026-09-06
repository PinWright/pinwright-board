---
id: F-world-precondition-on-mutating-verbs
title: "Mutating verbs have no caller-assertable world precondition, so a world swap between two calls silently writes into a different map — while render.capture_open_level already guards exactly this"
status: OPEN
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

## History
- `#1-filed` `OPEN` reporter — `effect.spawn_niagara` returned success with `mapPath: "/Game/FPS/Test/T_Player"` roughly three seconds after `level.load` had returned `activeLevelPath: "/Game/FPS/Test/T_VFX"`, because another stream loaded its own map and started PIE in between; the actor was created in a map belonging to a different stream and the response carried no error. No parameter exists to express "spawn only if the active world is still T_VFX", and most mutating verbs do not echo the world they acted on at all — while `render.capture_open_level` refuses `VIEWPORT_WORLD_MISMATCH` and publishes `activeWorld`/`viewportWorld`/`worldMatches` for the same hazard. Asking for an optional `expectedWorld` precondition plus an unconditional `activeWorld` echo on the world-writing verbs (`effect.spawn_niagara`, `actor.spawn`, `actor.spawn_batch`, `actor.delete`, `actor.set_transform`, `level.save`). Caught by reading `mapPath` and reverted with `actor.delete` (`existsAfter:false`), so no content was changed, but the foreign map was left dirty and a batched spawn-capture-delete would have scored one stream's review against another stream's level.
