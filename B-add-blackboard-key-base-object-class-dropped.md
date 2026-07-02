---
id: B-add-blackboard-key-base-object-class-dropped
title: "`ai.add_blackboard_key` silently drops `baseObjectClass` — Object keys are always created with BaseClass=UObject"
status: IN-REVIEW
severity: Medium
category: bug
tags: [ai, blackboard, add_blackboard_key, baseObjectClass, object-key, silent-noop]
---

# `ai.add_blackboard_key` ignores the documented `baseObjectClass` parameter

The wiki page for `ai.add_blackboard_key` documents `baseObjectClass` (`string`, optional): "Base class for Object keys". When you create an Object-typed key and pass `baseObjectClass`, the call returns success but the supplied class is **silently discarded**: the new key's `BlackboardKeyType_Object` subobject keeps its default `BaseClass = /Script/CoreUObject.Object` instead of the requested class. The success result also never echoes `baseObjectClass`, so nothing in the response hints that it was dropped.

This makes Object-typed Blackboard keys effectively unusable through the MCP for their intended purpose: you cannot constrain a key to `Actor`, `Pawn`, etc., and there is no other documented verb to set the base class afterward. The base class is real and meaningful — the auto-created `SelfActor` key on every fresh Blackboard correctly carries `BaseClass = /Script/Engine.Actor`; only the user-supplied value is ignored.

## Repro (verbatim, replayed via mcp__editor-automation__call)

```
call ai.create_blackboard_asset {name:"BB_ReplayConfirm", path:"/Game/AI/Blackboards"}
  -> success

call ai.add_blackboard_key {blackboardPath:"/Game/AI/Blackboards/BB_ReplayConfirm",
                            keyName:"TargetActor", keyType:"Object", baseObjectClass:"Actor"}
  -> {"keyIndex":1,"keyName":"TargetActor","keyType":"Object",...}   # success, no baseObjectClass echoed

call property.get {objectPath:"/Game/AI/Blackboards/BB_ReplayConfirm.BB_ReplayConfirm:BlackboardKeyType_Object_1",
                   propertyName:"BaseClass"}
  -> {"value":"/Script/CoreUObject.Object", ...}                    # WRONG — expected /Script/Engine.Actor
```

Also tried the fully-qualified form `baseObjectClass:"/Script/Engine.Pawn"` (key `TargetPawn`) — same result: `BlackboardKeyType_Object_2.BaseClass = /Script/CoreUObject.Object`. So it is not an input-format issue (short name vs class path); the parameter is simply never applied.

Contrast: the auto-generated `SelfActor` key on the same asset reads `KeyType.BaseClass = /Script/Engine.Actor` via `property.get Keys` and via `asset.dump`, confirming the field is real and that only the handler-supplied value is being dropped.

## What it should do

When `keyType:"Object"` (and `Class`) and `baseObjectClass` is supplied, the handler should resolve the class (accepting both a bare class name like `Actor` and a `/Script/...` path) and set it on the created `BlackboardKeyType_Object` (`UBlackboardKeyType_Object::BaseClass`) before marking the asset dirty / saving. The success response should also echo the resolved `baseObjectClass` so a caller can confirm it stuck. If the class cannot be resolved, return a clean error instead of silently falling back to `UObject`.

## History
- `#1-initial-repro` `OPEN` reporter — Found during a "guard AI data layer" task (seed `ai.create_blackboard_asset`). Replay-confirmed deterministically: `ai.add_blackboard_key {keyType:"Object", baseObjectClass:"Actor"}` returns success but the resulting `BlackboardKeyType_Object` subobject has `BaseClass=/Script/CoreUObject.Object`, not `/Script/Engine.Actor`; same with the fully-qualified `/Script/Engine.Pawn`. The auto-created `SelfActor` key correctly shows `BaseClass=/Script/Engine.Actor`, proving the field is meaningful and that the user-supplied value specifically is dropped. The documented `baseObjectClass` param (wiki: "Base class for Object keys") is therefore a silent no-op. Verified via `property.get` on the named `BlackboardKeyType_Object_N` subobjects and `asset.dump` (properties.json renders `KeyType:{}` for the broken key vs `KeyType:{BaseClass:.../Engine.Actor}` for SelfActor).
- `#2-fix` `IN-REVIEW` developer — Fixed the dropped parameter in `ai.add_blackboard_key`. In `Source/EditorAutomationRpcGateway/Private/Handlers/AI/AIHandler.cpp`, replaced the dead local + `// Could set base class here` stub with up-front resolution: when `keyType` is `Object`/`Class` and `baseObjectClass` is supplied, resolve it via `ResolveClassByName` (accepts a bare name like `Actor`/`Pawn` and a `/Script/...` path), and reject an unresolvable class with `CLASS_NOT_FOUND` *before* mutating the asset (no more silent fallback to `UObject`). The resolved `UClass*` is assigned to `UBlackboardKeyType_Object::BaseClass` (and the parallel `UBlackboardKeyType_Class::BaseClass`) before save, and the success response now echoes the resolved `baseObjectClass` (full path). Regression test `FAIAddBlackboardKeyAppliesBaseObjectClassTest` (`EditorAutomationRpcGateway.ai.add_blackboard_key.AppliesBaseObjectClass`) in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp` creates a real Blackboard, adds an Object key with `baseObjectClass:"Pawn"`, reloads the asset and asserts the saved key's `BaseClass == APawn::StaticClass()` (would be `UObject` if reverted), asserts the response echoes `/Script/Engine.Pawn`, and asserts an unresolvable class is rejected with `CLASS_NOT_FOUND` instead of fake-succeeding. Not yet compiled/run — verification is a later phase.
