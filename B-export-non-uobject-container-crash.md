---
id: B-export-non-uobject-container-crash
title: "Exporting an object-typed property from a non-UObject container crashes the editor"
status: IN-REVIEW
severity: Critical
category: bug
tags: [property-export, object-call-function, crash, reflection]
encounters: 1
lastSeen: 2026-09-03T10:50:21Z
---

# Exporting an object-typed property from a non-UObject container crashes the editor

`object.call_function` on a UFUNCTION whose **return value is a component-typed pointer** kills the
editor outright (`EXCEPTION_ACCESS_VIOLATION`, unsaved work lost). Reproduced twice in a row on
UE 5.8.

**Repro**

```
call("object.call_function", {
  objectPath: "<...>:PersistentLevel.<DroneActor>.CameraHub",
  function:   "GetActiveCamera"          // BlueprintPure, returns UCameraComponent*
})
```

The first attempt wedged the game thread (`EDITOR_NOT_READY`, "game thread inside PinWright RPC
'object.call_function'"), the second crashed straight out. `GetActiveCameraIndex` (returns `int32`)
and `SetCameraMode` (`void`) on the *same object* both answer normally, so it is specific to the
object-typed return. The same UFUNCTION called through `python.execute` returns fine, which rules
out the game code.

**Callstack**

```
UStruct::IsChildOf()                                    Class.cpp:2744
UObjectBaseUtility::IsA<UClass*>()                      UObjectBaseUtility.h:650
`anonymous namespace'::ShouldExpandObjectPropertyValue  Utils/PropertyExport.cpp:126
ExportPropertyToJsonValue                               Utils/PropertyExport.cpp:752
AutoHandler_314_                                        Handlers/Reflection/ObjectCallFunctionHandler.cpp:292
```

**Root cause**

`ExportPropertyToJsonValue(void* TargetContainer, FProperty*)` (`PropertyExport.cpp:605`) takes an
**untyped** container, but two paths inside it assume that container is a `UObject`:

- `PropertyExport.cpp:125-126` in `ShouldExpandObjectPropertyValue` —
  `const UObject* OwnerObject = static_cast<const UObject*>(TargetContainer); return OwnerObject && OwnerObject->IsA(OwnerClass) && ...`
- `PropertyExport.cpp:746-747` in the SCS-CDO fallback —
  `UObject* AsObject = static_cast<UObject*>(TargetContainer); if (AsObject && AsObject->HasAnyFlags(RF_ClassDefaultObject))`

`ObjectCallFunctionHandler.cpp:292` calls it as `ExportPropertyToJsonValue(Parms, Rec.Prop)`, where
`Parms` is the **`ProcessEvent` parameter frame** — a raw stack buffer, not a `UObject`. The
`static_cast` therefore produces a bogus `UObject*`, and `IsA()` dereferences its `ClassPrivate`
as a `UStruct`, which is the access violation.

The crashing branch is only reached for **component-typed** properties: `:112-115` sets
`bIsComponentProperty` when the value casts to `UActorComponent` and `PropertyClass` is a component
subclass, and only that branch runs the ownership check. A non-component object return takes the
`CPF_PersistentInstance | CPF_InstancedReference` flags branch at `:118` and survives — which is why
this only shows up on functions returning a component.

**Second latent instance (same defect, different caller)**

`PropertyExport.cpp:456` exports struct members with
`ExportPropertyToJsonValue(const_cast<void*>(StructPtr), Field)` — a **struct** container, also not a
`UObject`. A component-typed `UPROPERTY` inside a `USTRUCT` should crash the same way on any verb
that reaches it (`asset.dump`, `object.inspect`, ...). Not reproduced yet; worth a test.

The array-inner path at `:364` is safe — it passes `nullptr` explicitly and the helper returns
`false` on a null container.

## Fix

The root cause was treating an untyped property-container address as a `UObject*`. That was unsafe both for the reported non-null component ownership check (`IsA`) and for the broader null-object SCS fallback (`HasAnyFlags`), which could dereference a parameter frame or USTRUCT address for any hard object property.

`Source/PinWright/Private/Utils/PropertyExport.h` and `Source/PinWright/Private/Utils/PropertyExport.cpp` now carry an encapsulated `FPropertyExportSource`: `FromRaw` never supplies ownership, while `FromObject` and resolver-classified sources retain verified ownership through nested struct, optional, array, map, and set storage without casting those inner addresses to UObject. Hard references are read from their value address through `FObjectPropertyBase::GetObjectPropertyValue`. Owned instanced collection elements therefore keep authored expansion, while the same values reached from a raw parameter/USTRUCT source remain paths or null. Recursive export propagates `FPropertyExportResult` failures directly, canonical map keys continue to use `ExportTextItem_Direct` after strict key-shape validation, and only the outer loss-tolerant adapter creates the legacy unsupported marker.

`Source/PinWright/Private/Handlers/Reflection/ObjectCallFunctionHandler.cpp` uses strict raw-frame export and returns `UNSUPPORTED_PARAM_TYPE` after exactly-once cleanup. `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp`, `Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp`, and `Source/PinWright/Private/Handlers/Actor/ComponentHandler.cpp` preserve resolver-captured ownership. Coverage lives in `Source/PinWright/Private/Tests/Reflection/TestObjectCallFunctionHandler.cpp`, `Source/PinWright/Private/Tests/Utility/TestAssetDumpInstancedSubobjects.h`, and `Source/PinWright/Private/Tests/Utility/TestAssetDumpInstancedSubobjects.cpp`; the contract documentation is `docs/reflection-invocation.md`, and the implementation record is `.codex/plans/mcp-sprint-20260904-export-container.md` plus this ticket.

Regression IDs are `PinWright.object.call_function.ComponentReturnPath`, `PinWright.utils.property_utils.StructComponentContainer`, `PinWright.utils.property_utils.StrictMarkerShapedMap`, `PinWright.utils.property_utils.StrictNestedUnsupported`, `PinWright.property.get.OwnedComponentSerialization`, and the restored `PinWright.utils.property_utils.InstancedSubobjectRecursion` / `InstancedSubobjectClassDiscriminator`. Existing `PinWright.utils.property_utils.MapStructKeyExport` remains unchanged and continues to define canonical FGuid key text.

Left alone: `ErrorCodes.h`, build/config files, engine source, generated skills, and unrelated repository changes. No compile, automation, editor, or MCP run was performed for this source-only fix.

## History
- `#1-crash-on-component-return` `OPEN` reporter - "object.call_function on a UFUNCTION returning UCameraComponent* crashes the editor (UStruct::IsChildOf via ShouldExpandObjectPropertyValue, PropertyExport.cpp:126). Container is the ProcessEvent parameter frame, static_cast<UObject*> on it is invalid. Hit twice on UE 5.8 while driving a PIE session; python.execute on the same function works. Same unchecked cast at PropertyExport.cpp:746 and a likely second instance via the struct-member path at :456."
- `#2-safe-raw-container-export` `IN-REVIEW` developer — "Added encapsulated FromRaw/FromObject export sources with nested owner provenance, compile-safe shared-ref results, direct support propagation through owned subobjects and recursive containers, canonical exact map-key text with unsupported-key rejection, safe hard-object reads, strict call-function cleanup/errors, known-owner caller propagation, and coverage for component returns, struct containers, marker-shaped maps, nested unsupported values, property.get ownership, owned instanced collections, and raw-source paths."
