---
id: B-spawn-gate-refuses-on-stale-outstanding-compile-flag
title: "effect.spawn_niagara refuses SYSTEM_NOT_COMPILED forever on a stale hasOutstandingCompilationRequests flag while every script in the system reads NCS_UpToDate"
status: OPEN
severity: High
category: bug
tags: [effect, spawn-niagara, niagara, compile, gate, false-negative, blocks-capture, stale-flag, vfx]
encounters: 1
lastSeen: 2026-09-07T06:47:00Z
---

# A compiled system cannot be spawned, and retrying does not clear it

`effect.spawn_niagara`'s pre-spawn readiness gate refuses with `SYSTEM_NOT_COMPILED` whenever the
system reports `hasOutstandingCompilationRequests: true`. That flag can be **stuck true on a system
whose scripts are all compiled**, and when it is, the gate refuses permanently: the system becomes
unspawnable and there is no verb that clears it.

Measured 2026-09-07, UE 5.8, gateway 27145, freshly restarted editor after a PinWright rebuild,
world `/Game/FPS/Test/T_VFX`, VFX critic holding the world lock.

## What I called

    call("effect.spawn_niagara", {systemPath: "/Game/FPS/VFX/NS_Tracer",
                                  location: {x:1000, y:0, z:140}, rotation: [0,90,0],
                                  name: "VC_B7", autoDestroy: false})

    -> SYSTEM_NOT_COMPILED
       "Niagara system '/Game/FPS/VFX/NS_Tracer' is not compiled and ready to spawn
        (compileStatus=outstanding)"
       compileStatus: "outstanding"
       compileWaited: false
       compileWaitedMs: 0.0009015202522277832
       compileTimedOut: false
       outstandingCompilationRequests: true
       compile.valid: true
       compile.hasOutstandingCompilationRequests: true
       compile.hasActiveCompilations: false

Retried once after a 6-second wait: byte-identical refusal, `compileWaitedMs`
0.0024028122425079346.

## Why the refusal is wrong

The same response carries the per-script compile state, and **not one script is unready**:

| script | compileStatus |
|---|---|
| `SystemSpawnScript` | `NCS_UpToDate` |
| `SystemUpdateScript` | `NCS_UpToDate` |
| `Tracer.SpawnScript` | `NCS_UpToDate` |
| `Tracer.UpdateScript` | `NCS_UpToDate` |
| `Tracer.EmitterSpawnScript` | `null` (uninitialised, the documented on-demand-compile state) |
| `Tracer.EmitterUpdateScript` | `null` (same) |

`compile.valid` is `true`. `hasActiveCompilations` is `false` — nothing is compiling. So the gate is
refusing on `hasOutstandingCompilationRequests` alone, a request that is outstanding and will never
become active, while every script it would need is already `NCS_UpToDate`.

Twelve other systems in the same folder spawned successfully in the same slot, seconds apart
(`NS_Muzzle_AR`, `NS_Muzzle_Pistol`, `NS_Blood`, `NS_Impact_Concrete`, `_Metal`, `_Wood`, `_Dirt`,
`_Glass`, `_Water`, `NS_Explosion`, `NS_Smoke_Grenade`, `NS_ShellEject`), all reporting
`compileStatus: "passed"`. `NS_Tracer` is the only one carrying the stuck flag, and it is the only
one that cannot be spawned.

## What I expected

Either the gate to consider the per-script state it already fetched and admit a system whose scripts
are all `NCS_UpToDate`, or — if an outstanding request genuinely must block — a bounded wait that
actually waits, and a typed way to clear or re-issue the request. Today the caller has neither: the
error names no remedy and no verb in the namespace resets the flag.

## Cost

It cost the VFX round-4 review its tracer capture. Every other system in the package was captured in
the same slot; `NS_Tracer` alone could not be placed, so the one system that the previous review
scored 7/10 goes unverified this round and its score has to be carried forward with a caveat. In a
build pipeline the same shape would be worse: a system that validates clean, compiles clean and
reports `valid: true` simply cannot be instantiated, with no diagnostic pointing at the cause.

## Relationship to `B-niagara-compile-wait-does-not-wait`

That ticket's `#12`/`#13` describe an engine-owned bounded wait landed in wave 10. This gate's wait
did not run: `compileWaited: false` with a measured budget of **0.0009 ms** and
`compileTimedOut: false`. I am not asserting the two share code — I have not read the source. If
they do, the wave-10 fix does not hold at this call site; if they do not, this is a second site that
needs the same treatment. I have returned that ticket to `OPEN` with a `#14-returned` entry carrying
the same measurements, cross-referencing this id.

## Workaround

None found from the RPC surface. `effect.spawn_niagara` is the only documented route to place a
Niagara system for a level capture, and rule 10 forbids capturing Niagara through its asset editor,
so a system in this state cannot be reviewed at all. Opening the asset in the Niagara editor would
presumably populate compile state and clear the request, but that is exactly what PLAN rule 10
forbids for Niagara.

## Root-cause guess

Not read from source. The observable is a system-level `bool` that latches true and is never
cleared, most likely because the request was enqueued and then satisfied (or discarded) by a path
that does not decrement it — plausibly across the plugin rebuild and editor restart that immediately
preceded this session, since the flag survived a fresh editor start with the asset loaded from disk.

## History
- `#1-filed` `OPEN` reporter — `effect.spawn_niagara` refused `SYSTEM_NOT_COMPILED (compileStatus=outstanding)` for `/Game/FPS/VFX/NS_Tracer` on two attempts six seconds apart in a freshly restarted editor, while the same response reported `compile.valid: true`, `hasActiveCompilations: false`, and all four initialised scripts at `NCS_UpToDate` (the two remaining at the documented `null` on-demand state). Twelve sibling systems in the same folder spawned successfully in the same slot with `compileStatus: "passed"`. The gate appears to refuse on `hasOutstandingCompilationRequests` alone, a flag that is stuck true and that no verb in the namespace can clear, making the system permanently unspawnable and therefore — under PLAN rule 10, which forbids capturing Niagara through the asset editor — permanently unreviewable. Its bounded wait did not run (`compileWaited:false`, `compileWaitedMs:0.0009`, `compileTimedOut:false`); see `B-niagara-compile-wait-does-not-wait` `#14-returned`.
