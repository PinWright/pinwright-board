---
id: B-level-get-info-actorcount-counts-destroyed-actors
title: "level.get_info actorCount keeps counting actors that actor.delete has already destroyed, so it cannot be used to verify cleanup"
status: OPEN
severity: Medium
category: bug
tags: [level, get_info, actor, delete, actorCount, stale-data, cleanup-verification, world-lock, vfx]
encounters: 1
lastSeen: 2026-09-05
---

# `actorCount` disagrees with `actor.list` by exactly the number of actors just deleted

`level.get_info`'s `actorCount` counts destroyed-but-not-yet-collected actors. After a
spawn/delete cycle it reads high by exactly the number of actors deleted, while every other verb
correctly reports them gone. An agent using it to prove it left a shared map clean gets a number
that says otherwise.

Measured 2026-09-05, UE 5.8, gateway 27145, `/Game/FPS/Test/T_VFX`, editor world (no PIE).

## What I called

    call("level.load", {levelPath: "/Game/FPS/Test/T_VFX"})
    sleep 2.5
    call("level.get_info", {})                     -> {"actorCount": 39}

    # eight times, one Niagara system each:
    call("effect.spawn_niagara", {systemPath: "...", name: "VC_S<n>", autoDestroy: false})
    call("effect.step_and_capture", {systemName: "VC_S<n>", ...})
    call("actor.delete", {actorName: "VC_S<n>"})
      -> {"success": true, "deletedCount": 1, "deleted": ["VC_S<n>"], "existsAfter": false}

    call("level.get_info", {})                     -> {"actorCount": 47}

## What happened

`actorCount` went **39 → 47** across a sequence whose net actor change is **zero**. The delta, 8,
is exactly the number of actors spawned and then deleted. Two independent verbs disagree with it,
called immediately afterwards in the same tick window:

| check | result |
|---|---|
| `level.get_info` `actorCount` | **47** (39 at load) |
| `actor.list {filter:"VC_", matchMode:"contains"}` | `{"actors": [], "count": 0, "totalMatches": 0}` |
| `effect.cleanup {filter:"VC_"}` | `{"removedActors": [], "removed": 0}` |
| each `actor.delete`'s own `existsAfter` | `false`, eight times |

So the actors really are gone — `actor.delete` is correct and honest, and `actor.list` agrees with
it. Only `actorCount` still counts them. The obvious mechanism is that it measures the raw level
actor array (or a `TActorIterator` without a pending-kill filter) and includes entries marked
`PendingKill`/`Garbage` that GC has not yet reaped; nothing in this session forced a GC between the
deletes and the read.

## What I expected

`actorCount` to return to 39, matching `actor.list`, `effect.cleanup` and `existsAfter`. Failing
that, a documented statement that the count includes pending-destruction actors, so callers know
not to use it as a cleanup check.

## Why it matters here

`Docs/fps/PLAN.md` rule 3 requires every world-lock holder to hand the map back clean, and
`actorCount` before/after is the natural way to prove it. It is the wrong tool and it fails in the
direction that wastes time: it says junk remains when the map is clean. The inverse would be worse —
this ticket does not establish which way it fails when actors genuinely do leak, because I never had
a leak to test against, and I am not claiming it would mask one.

It also interacts with a rule the same PLAN imposes: agents are told to confirm `level.get_info`
after a load as the settle probe for `render.capture_open_level` (rule 11). That use is fine — it is
only the `actorCount` field that is unreliable.

## Workaround

Verify cleanup with `actor.list` against your own label prefix, or with `actor.delete`'s
`existsAfter` / `deleted[]`, and treat `actorCount` as an upper bound only. I used
`actor.list {filter: "<prefix>", matchMode: "contains"}` and required `count: 0`.

## Root-cause guess

Not read from source; the delta matching the spawn count exactly is the whole basis. Candidate is an
unfiltered actor enumeration behind `get_info`, where `actor.list` applies `IsValid` /
`!IsPendingKillPending` and `get_info` does not.

## History
- `#1-filed` `OPEN` reporter — `level.get_info` returned `actorCount: 47` for `/Game/FPS/Test/T_VFX` immediately after a sequence of eight `effect.spawn_niagara` + `actor.delete` pairs whose net actor change is zero; the level had read `actorCount: 39` after `level.load` and before any spawn, so the count was high by exactly the eight actors that had just been deleted. Each `actor.delete` reported `deletedCount: 1`, `existsAfter: false`; `actor.list {filter:"VC_", matchMode:"contains"}` returned `count: 0` and `effect.cleanup {filter:"VC_"}` returned `removed: 0`, both called seconds later, so the actors were genuinely destroyed and only `actorCount` still counted them. No PIE, no GC forced between the deletes and the read. Filed by the VFX critic while verifying it had handed the shared test map back clean under PLAN rule 3; the practical cost is that the obvious cleanup check reports dirty on a clean map. Workaround in use: verify with `actor.list` on the label prefix and require `count: 0`.
