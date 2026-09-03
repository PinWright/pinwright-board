---
id: B-effect-spawn-false-success
title: "effect.spawn_niagara can return success for a non-Niagara asset and silently drops requested attachment and auto-destroy"
status: OPEN
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

## Related

- `E-effect-actor-name-slot-vs-actorname`

## History
- `#1-source-scan` `OPEN` reporter -- Confirmed all three success fall-throughs in current source;
  no actor was spawned during this read-only scan.
