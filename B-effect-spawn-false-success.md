---
id: B-effect-spawn-false-success
title: "effect.spawn_niagara can return success for a non-Niagara asset and silently drops requested attachment and auto-destroy"
status: IN-REVIEW
severity: High
category: bug
tags: [effect, spawn-niagara, false-success, asset-class, attachment, auto-destroy]
---

# Success does not require the requested Niagara state to exist

`effect.spawn_niagara` checks only `UEditorAssetLibrary::DoesAssetExist(systemPath)`, then loads the
asset as `UObject` (`EffectHandler.cpp:1204-1254`). It spawns an `ANiagaraActor` first and assigns
the component asset only inside `if (NiComp && NiagObj->IsA<UNiagaraSystem>())` (`:1264-1270`). A
Material, Texture, or any other existing asset therefore produces an empty Niagara actor and still
reaches `SendSuccess`.

The same handler parses `autoDestroy` but never applies it to the component (`:1233-1235`). If
`attachToActor` names no actor, the lookup simply leaves `Parent=null` and the spawn still succeeds
unattached (`:1272-1289`). None of those unapplied conditions is represented in the response;
`AddActorVerification` proves only that an actor exists.

## What it should do

Load and type-check `UNiagaraSystem` before spawning. Treat a requested missing parent as a typed
error (or return an explicit attachment warning/status), apply `SetAutoDestroy`, and verify the
component's assigned system, attachment parent, and auto-destroy state before success. If a
post-spawn invariant fails, destroy the newly created actor before returning the error.

## Workaround

Pre-validate the asset class and parent label, then inspect the spawned component/attachment rather
than relying on actor existence.

## Fix

Root cause: the handler treated actor creation as the terminal success condition. Asset assignment,
auto-destroy, attachment, activation, and compile health were either conditional, ignored, or never
read back.

Changed `Source/PinWright/Private/Handlers/VFX/EffectHandler.cpp` to type-check and preflight the
requested parent in the same editor/PIE target world before spawning, use `NiagaraCompileWait` plus
the shared post-wait engine-readiness predicate, prepare the component while auto-activation is
disabled, and activate exactly once after applying the advertised state and attachment. It verifies the assigned
system, attachment, auto-destroy, and `IsActive()` before success. Explicit compile failures and
systems that remain unready after the bounded wait return
`SYSTEM_NOT_COMPILED`; activation/state failures return `EFFECT_NOT_ACTIVE`, with measured state in
both response paths. Failed post-spawn invariants destroy the new actor through its `UWorld`, which
also supports PIE actors. Added both codes to `Handlers/ErrorCodes.h`.

Behavioral coverage: `PinWright.effect.spawn_niagara.MeasuredStateAndCompileFailure` duplicates the
stock Niagara fixture into a transient package, invokes the registered handler, checks the response
against the live component/parent/auto-destroy state, uses a test-only post-activation seam to force
the real component inactive and verify `EFFECT_NOT_ACTIVE` cleanup leaves no actor or component,
injects `NCS_Error` into an owned script, and checks the typed rejection creates no second actor.
Fixtures use strong object ownership rather than raw root manipulation. Updated
`docs/wiki-src/effect.md` with the new success and error contract.

Deliberate non-changes: actor-label ambiguity and rotation/scale input-shape validation remain
outside this ticket. No build, editor run, or automation run was performed under this source-only
task's constraints.

## Related

- `E-effect-actor-name-slot-vs-actorname`

## History
- `#1-source-scan` `OPEN` reporter -- Confirmed all three success fall-throughs in current source;
  no actor was spawned during this read-only scan.
- `#2-verified-spawn-state` `IN-REVIEW` developer -- Type-checked and preflighted inputs, applied and
  read back the requested Niagara state, gated success on bounded compile and `IsActive()` checks,
  added transient-system behavioral coverage, and documented the measured response contract.
- `#3-verifier-repair` `IN-REVIEW` developer -- Delayed Niagara asset assignment until the component
  was unregistered with auto-activation disabled, resolved parents in the spawn target world, made
  cleanup PIE-safe, and added behavioral coverage for typed inactive-state cleanup without raw roots.
- `#4-shared-compile-readiness` `IN-REVIEW` developer -- Replaced the script-status-only spawn gate
  with the shared bounded-wait readiness predicate: explicit failures still refuse, while an
  engine-ready quiet system is no longer rejected because on-demand script diagnostics are unverified.
