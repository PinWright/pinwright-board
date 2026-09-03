---
id: B-export-non-uobject-container-crash
title: "Exporting an object-typed property from a non-UObject container crashes the editor"
status: OPEN
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

**Suggested fix**

The container's type has to stop being guessed. Either give `ExportPropertyToJsonValue` an explicit
`UObject* OwnerObject` parameter alongside the raw container (call sites know which they have:
`ObjectCallFunctionHandler` and the struct path pass `nullptr`, object paths pass the object), or
carry a `bContainerIsUObject` flag through. Both `static_cast<UObject*>(TargetContainer)` sites then
key off that instead of the pointer. A guard that only null-checks is not enough — the pointer is
non-null and garbage.

Worth covering with a test for each container shape: UObject, function parameter frame, struct.

## History
- `#1-crash-on-component-return` `OPEN` reporter - "object.call_function on a UFUNCTION returning UCameraComponent* crashes the editor (UStruct::IsChildOf via ShouldExpandObjectPropertyValue, PropertyExport.cpp:126). Container is the ProcessEvent parameter frame, static_cast<UObject*> on it is invalid. Hit twice on UE 5.8 while driving a PIE session; python.execute on the same function works. Same unchecked cast at PropertyExport.cpp:746 and a likely second instance via the struct-member path at :456."
