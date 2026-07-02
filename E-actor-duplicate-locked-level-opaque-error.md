---
id: E-actor-duplicate-locked-level-opaque-error
title: "actor.duplicate on a locked level returns bare [DUPLICATE_FAILED] Failed to duplicate actor, naming no cause and hiding the level.set_locked recovery"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [actor, duplicate, locked-level, error-message, diagnostic, discoverability]
encounters: 2
lastSeen: 2026-07-02T03:26:06.4106887+03:00
---

# `actor.duplicate` hides the locked-level cause behind a bare `[DUPLICATE_FAILED]`

When the target actor lives on a **locked** level (the persistent level here,
locked via `level.set_locked locked:true`), `actor.duplicate` rejects the call
with the bare error:

> `[DUPLICATE_FAILED] Failed to duplicate actor`

The message names only the *symptom* (the duplicate didn't happen) and gives the
caller no hint of the *cause* (the level is locked) nor the *recovery* (unlock it
via `level.set_locked locked:false`). This is correct-but-misleading behavior: a
locked level legitimately blocks duplication, so the failure itself is right —
but the wording strands the caller. It is the same "error names a symptom, not
the cause, and drives trial-and-error" class already filed as the OPEN
`E-landscape-edit-extent-error-not-diagnostic` (bare `[INVALID_LANDSCAPE] Failed
to get landscape extent`) and `E-level-load-file-not-found-vs-in-memory-orphan`,
here in the `actor` namespace.

## Root cause (engine path)

`actor.duplicate` (LifecycleHandler.cpp, the `actor.duplicate` handler) calls
`UEditorActorSubsystem::DuplicateActor(Found, World, Offset)` and, when it
returns null, emits the generic error unconditionally:

```cpp
AActor *Duplicated = ActorSS->DuplicateActor(Found, Found->GetWorld(), Offset);
if (!Duplicated) {
  Ctx.SendError(TEXT("DUPLICATE_FAILED"), TEXT("Failed to duplicate actor"));
  return true;
}
```

Under the hood `DuplicateActor` -> `GEditor->DuplicateActors` -> `UEditorActor`
machinery early-returns with **no duplicated actor** when the destination level
is locked (`FLevelUtils::IsLevelLocked` guard in the engine's `EditorActor.cpp`
duplicate-to-level path). The handler has no visibility into *why* the engine
returned null, so it collapses every failure mode — locked level, read-only
level, no current level, transient engine refusal — into one opaque string.

## Why it matters — the process friction

In the audited task ("spawn one source pillar, then duplicate it four times down
a corridor") the opaque error drove a multi-step flail and a source dive that a
diagnostic message would have collapsed into one read. From the attempt's call
log, `actor.duplicate` failed **4 times** with the identical bare
`[DUPLICATE_FAILED]` while the caller chased hypotheses the error never ruled
out:

- retried after `actor.select` (thought the source needed selecting; also hit a
  separate `[MISSING_REQUIRED_PARAM]` because `select` wants `actorNames` array),
- retried after `editor.stop` (thought a stray PIE session was blocking it —
  `alreadyStopped`),
- retried with **no offset** as a diagnostic (to rule out the translation),
- retried by the internal name `StaticMeshActor_0` (to rule out label
  resolution),
- ran `actor.describe`, `actor.find_by_class WorldSettings`, and
  `property.get bEnableWorldPartition` (which itself errored `[PROPERTY_NOT_FOUND]`)
  to reverse-engineer the world/level state.

The friction note records the cost directly: the agent "had to read the plugin
handler + UE engine source (EditorActor.cpp DuplicateActorsToLevel) to discover
the real cause was a LOCKED persistent level (IsLevelLocked early-return), then
unlock it via level.set_locked before duplicate worked." Only after
`level.set_locked locked:false` did all four duplicates succeed. One source dive
plus ~8 wasted RPCs to learn what one error string could have stated.

## Replay confirmation (this filing)

Reproduced live against `mcp__editor-automation__call` on
`/Game/Maps/ExampleProjectWelcome`:

1. `actor.spawn` a Cube StaticMeshActor `ReplayPillar_Source` at {0,0,0} — ok.
2. `level.set_locked {levelPath:/Game/Maps/ExampleProjectWelcome, locked:true}` — ok.
3. `actor.duplicate {actorName:ReplayPillar_Source, offset:{x:300}, newName:ReplayPillar_1}`
   -> `[DUPLICATE_FAILED] Failed to duplicate actor` (verbatim, no cause).
4. `level.set_locked {... locked:false}` — ok.
5. The **identical** `actor.duplicate` call now **succeeds**
   (`{"source":"ReplayPillar_Source","actorName":"ReplayPillar_1", ... "offset":[300,0,0]}`).

The lock is the sole variable between failure and success, proving the bare
error hides a fully recoverable, caller-fixable cause.

## Fix (message clarity only — no behavior change required)

In the `actor.duplicate` null-result path (LifecycleHandler.cpp), before sending
the generic `DUPLICATE_FAILED`, check the source actor's level lock state and, if
locked, name the cause and the recovery, for example:

> `[LEVEL_LOCKED] Cannot duplicate '<label>': its level
> '<level package>' is locked. Unlock it first via
> level.set_locked {levelPath:'<level package>', locked:false}.`

(Use `FLevelUtils::IsLevelLocked(Found->GetLevel())` for the check.) Keep the
generic `DUPLICATE_FAILED` only as the fallback for genuinely unexplained null
results. The same lock pre-check would help the sibling mutators that route
through the editor-actor subsystem and silently fail on a locked level
(e.g. `actor.set_transform`, `actor.delete`), but `actor.duplicate` is the
demonstrated case here.

## History
- `#1-initial-repro` `OPEN` reporter — Struggle-audit of a corridor-pillar
  blockout (`actor.duplicate` seed). `actor.duplicate` failed 4x with the
  identical bare `[DUPLICATE_FAILED] Failed to duplicate actor` because the
  persistent level was locked; the error named no cause, driving retries after
  `actor.select` / `editor.stop` / no-offset / internal-name plus
  `actor.describe`, `actor.find_by_class`, `property.get` diagnostics and a dive
  into engine EditorActor.cpp before the agent found `IsLevelLocked` and
  unlocked via `level.set_locked`. Replay-confirmed live: same `actor.duplicate`
  call errors with the level locked and succeeds with it unlocked (only the lock
  changed). Correct-but-misleading wording — the failure is right, the message
  strands the caller. Same symptom-not-cause class as the OPEN
  `E-landscape-edit-extent-error-not-diagnostic` /
  `E-level-load-file-not-found-vs-in-memory-orphan`, different namespace. Dedup:
  ripgrep across OPEN/closed board files found no existing ticket on
  `actor.duplicate` error wording or locked-level duplication (qmd unavailable).
- `#2-level-locked-precheck` `IN-REVIEW` developer — Added a locked-level
  pre-check to the `actor.duplicate` null-result path in
  `Source/EditorAutomationRpcGateway/Private/Handlers/Actor/LifecycleHandler.cpp`
  (includes `Engine/Level.h` + `LevelUtils.h`). Before calling
  `UEditorActorSubsystem::DuplicateActor`, it tests
  `FLevelUtils::IsLevelLocked(Found->GetLevel())` (the source level the engine's
  `DuplicateActorsToLevel` guards on at `EditorActor.cpp:483`) and, when locked,
  emits `[LEVEL_LOCKED] Cannot duplicate '<label>': its level '<package>' is
  locked. Unlock it first via level.set_locked {levelPath:'<package>',
  locked:false}.` instead of the bare `DUPLICATE_FAILED`; the generic
  `DUPLICATE_FAILED` remains the fallback for any other unexplained null. No
  behavior change — message clarity only. Regression test added:
  `EditorAutomationRpcGateway.actor.duplicate.LockedLevelDiagnostic` in
  `Source/EditorAutomationRpcGateway/Private/Tests/World/TestActorHandlers.cpp`
  spawns a real actor into the editor world, locks its level via
  `FLevelUtils::ToggleLevelLock`, invokes the production `actor.duplicate`
  handler and asserts the `LEVEL_LOCKED` code (would be `DUPLICATE_FAILED` if the
  pre-check were reverted), then unlocks and confirms the identical call
  succeeds (lock is the sole variable) and restores the level's prior lock
  state.
- `#3-additional-spawn-duplicate-lock-inconsistency` `OPEN` reporter —
  Struggle-audit of a park-bollard-along-a-spline task (focus
  `geometry.duplicate_along_spline`; 42 calls; outcome done). TWO signals: (a)
  **Liveness / fix-confirmation** — the `#2` `[LEVEL_LOCKED]` diagnostic FIRED in
  the wild: the agent's first `actor.duplicate` batch on the default-locked
  `/Game/Maps/ExampleProjectWelcome` failed with the verbatim improved message
  ("Cannot duplicate 'Bollard': its level '…ExampleProjectWelcome' is locked.
  Unlock it first via level.set_locked {levelPath:'…', locked:false}."), the
  agent read the named recovery and recovered instantly (`level.set_locked
  locked:false` → the identical batch succeeded). The message-clarity fix works
  as intended — no more opaque `DUPLICATE_FAILED` flail. (b) **New residual angle
  the message fix does NOT address — spawn/duplicate lock inconsistency:** in the
  same trace `actor.spawn` (Cylinder "Bollard") and `spline.create_spline_actor`
  BOTH succeeded into that same locked persistent level with no lock complaint,
  yet the immediately-following `actor.duplicate` of that just-spawned actor was
  blocked by the lock. So the lock is enforced by `duplicate` but ignored by
  `spawn`/`create_spline_actor` — a caller who just spawned successfully has no
  reason to expect duplicate to be gated, and here it cost a wasted 8-call
  duplicate batch (all `LEVEL_LOCKED`) before the unlock+retry. This is a
  behavioral-consistency gap distinct from the wording fix: either the whole
  spawn/duplicate family should honor the editor lock or none should (or
  `actor.duplicate` should auto-unlock-then-relock like spawn tolerates). Low
  severity, self-correcting (excellent error), but a real per-batch cost. Logged
  here rather than as a near-duplicate file because it shares the `locked-level` +
  `actor.duplicate` family; consider before closing the message-only fix.
