---
id: B-niagara-emitter-save-guard-deadlocks-unused-emitter
title: "niagara edit save:true can never succeed on a standalone emitter asset that no loaded system inherits — the compile guard demands a compile niagara.compile refuses to request, and asset.save walks straight past the same guard"
status: OPEN
severity: Medium
category: bug
tags: [niagara, emitter, save, compile, compile-guard, deadlock, set_module_input, set_static_switch, asset-save, inconsistent-gate]
encounters: 2
lastSeen: 2026-09-05T20:59:00+03:00
---

# The save guard on `niagara.*` edits is unsatisfiable for an emitter asset no loaded system uses

`niagara.set_module_input {save: true}` (and `set_static_switch`, same path) on a **standalone
`UNiagaraEmitter` asset** refuses the save when it cannot observe a completed compile:

```
LogPinWrightSubsystem: Warning: niagara edit on '/Game/FPS/VFX/Emitters/E_ImpactWater_Droplets':
no requested compile was observed complete (waited 0.0 s); refusing to save an invalidated compile.
Re-run niagara.compile then asset.save.
```

The remedy the message names does not exist for this asset. Niagara exposes compile queues only on
**systems**, so `niagara.compile` on an emitter is a request routed through the loaded systems that
use it. When none does, it requests nothing:

```
call("niagara.compile", {assetPath:"/Game/FPS/VFX/Emitters/E_ImpactWater_Droplets",
                         force:true, wait:true, timeoutSeconds:60})
-> {"requested":false,"compiled":false,"completed":false,"waited":false,"waitedMs":0.0007,
    "timedOut":false,"stillCompiling":[],"outstandingCompilationRequests":false,
    "status":"notRequested","durationMs":0.034}
```

`force: true` does not help — the emitter is not the thing the engine compiles. `compile_status`
documents the same shape from the other side ("For a standalone emitter with
`affectedSystemCount: 0`, it remains unverified"). So the guard's precondition is not merely unmet,
it is **unreachable**: `save: true` on such an emitter fails every time, forever, and the failure
text sends the caller round a loop that cannot terminate.

## The same call succeeds through `asset.save`, which makes the guard advisory rather than a guard

```
call("asset.save", {assetPath:"/Game/FPS/VFX/Emitters/E_ImpactWater_Droplets", force:true})
-> {"saveRequested":true,"saved":true,"saveState":"written",
    "saveDetail":"This call wrote the .uasset; the asset is durable on disk.","sizeBytes":96488}
```

Verified on disk, not from the response: mtime moved to 2026-09-05 20:49:55 +0300, and the
length-prefixed pin-default FStrings in the package changed exactly as written — `"760.0"` 1 -> 0,
`"240.0"` 0 -> 1, `"(X=1.0,Y=0.0,Z=0.0)"` 1 -> 0, `"(X=0.0,Y=0.0,Z=1.0)"` 0 -> 1, `"2.0"` 1 -> 0
(`Cone Axis Coordinate Space` Local -> World, which serialises as a bare `"1.0"` and is otherwise
invisible to a grep).

Two verbs in the same plugin therefore disagree about whether the identical package is safe to
write, and the one with the safety reasoning attached is the one that can be bypassed by a caller
who reads the error text and does what any caller would do next. Whatever corruption the guard
exists to prevent, `asset.save` does not prevent it.

## Why it matters

This is not an exotic asset shape. Every emitter asset in a project built with the pre-rebuild
`niagara.add_emitter` is in exactly this state, because that verb produced **unlinked snapshots**
(`B-niagara-add-emitter-snapshots-emitter-silently`): the systems hold copies with no parent, so no
loaded system "uses" the emitter asset in the sense `compile` measures, and every edit to the asset
hits the deadlock. Measured on `/Game/FPS/VFX/NS_Impact_Water` — all five handles report
`versionedEmitterData.parent = {"inherited": false}`.

The practical outcome is that a caller learns to route every emitter edit through `asset.save`, which
is precisely the habit the guard was added to prevent.

## Suggested fix (not source-confirmed; diagnosis is from RPC responses and the editor log only)

1. **Make the guard's own precondition reachable, or drop the guard on this path.** When the target
   is an emitter and `niagara.compile` would return `notRequested` because `affectedSystemCount == 0`,
   there is no compiled system state to invalidate — the emitter's scripts are compiled when a system
   next pulls them in. Either save, or refuse with an error that names the real situation
   (`EMITTER_NOT_COMPILABLE_STANDALONE`) instead of instructing a loop.
2. **If the refusal is correct, `asset.save` must honour it too.** A gate that one verb enforces and
   its documented sibling ignores is worse than no gate: it teaches the bypass.
3. The log line should say which of the two it is. "Re-run `niagara.compile` then `asset.save`" is
   actionable-sounding advice whose first half is a no-op and whose second half defeats the check.

## Cross-ref

- `B-niagara-add-emitter-snapshots-emitter-silently` — the reason a project ends up full of emitter
  assets that no loaded system inherits. Its rebuild fix (`inherit` defaulting true) prevents new
  ones; it does not re-parent existing handles, so pre-rebuild systems stay in this state.
- `B-niagara-save-no-disk-write` — the opposite failure on the same surface (claimed a write that
  never happened); this one refuses a write that succeeds through another door.
- `B-niagara-compile-wait-does-not-wait` — same verb, different failure.

## History

- `#1-initial-repro` `OPEN` builder — Found during the water-emitter reconcile on
  EAContentExamples58 (UE 5.8, live editor port 27145, 2026-09-05), applying the system's
  `AddVelocityInCone` edits forward onto `/Game/FPS/VFX/Emitters/E_ImpactWater_Droplets` and the
  `ShapeLocation` shape switch onto `E_ImpactWater_Crown`. Both responses, the log line and the
  `asset.save` result above are verbatim; the byte counts were read from the `.uasset` after the
  save, not from the response. No source was read.
- `#2-independent-repro-blood-emitters` `OPEN` reporter — Independent second encounter the same day
  from a different stream (the `NS_Blood` cone-axis fix), on three more standalone emitters:
  `/Game/FPS/VFX/Emitters/E_Blood_{Drips,Spray,Mist}`. `niagara.compile {assetPath:
  "/Game/FPS/VFX/Emitters/E_Blood_Drips", force:true, wait:true, timeoutSeconds:60}` returned
  `{"requested":false,"compiled":false,"completed":false,"waited":false,"waitedMs":0.0023,
  "timedOut":false,"stillCompiling":[],"outstandingCompilationRequests":false,
  "status":"notRequested","durationMs":0.147}` — the shape in the body, reproduced verbatim on a
  different asset. `asset.save {force:true}` then wrote all three
  (`saveState:"written"`, sizes 102933 / 104300 / 99972), verified against the file: fresh mtimes
  (2026-09-05 20:55:12 / :15 / :20 +0300) and a git-LFS before/after string diff of each `.uasset`
  showing `"(X=1.0,Y=0.0,Z=0.0)"` 0 -> 1, `"2.0"` +1, and the two override-pin names
  `AddVelocityInCone.Cone Axis` / `AddVelocityInCone.Cone Axis Coordinate Space` entering the name
  table. So `#1`'s central claim — the guard is unreachable on this class of asset while
  `asset.save` writes it anyway — reproduces on assets from a different system, and this stream
  reached the same workaround independently (never pass `save:true`; use `asset.save`).
  One detail to add to fix point 3: the emitter-path `niagara.compile` also emits
  `LogUObjectGlobals: Warning: Failed to find object 'NiagaraSystem /Game/FPS/VFX/Emitters/E_Blood_Drips'`
  into the editor log before returning its clean `notRequested` — the resolver evidently tries
  `UNiagaraSystem` first and lets the miss log. Harmless, but it puts a scary "Failed to find object"
  line in the log for a call that succeeded, which costs a reader time when triaging an unrelated
  failure in the same window. Verbatim from `Saved/Logs/EAContentExamples58.log` line 4402,
  `[2026.09.05-17.53.20:231]` (log is UTC; machine is UTC+3). No source was read this session.
