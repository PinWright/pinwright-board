---
id: B-probe-subobject-handle-ignores-class
title: "blueprint.probe_subobject_handle echoes its componentClass param unvalidated — returns success:true for a nonexistent/non-component class"
status: IN-REVIEW
severity: Medium
category: bug
tags: [blueprint, subobject, scs, silent-success, ignored-param, validation]
---

# probe_subobject_handle echoes componentClass back unvalidated — a bogus/non-component class returns success:true

`blueprint.probe_subobject_handle` takes an optional `componentClass`
(`string`, "Component class to probe (default StaticMeshComponent)"). The
handler reads it into a local string and only **echoes it back** in the
result — it never resolves the string to a `UClass` and never validates it.

The scope here is the **minimum honesty fix**, not a new feature. The method
is a `USubobjectDataSubsystem` availability/smoke probe whose handles come
from an empty temp Blueprint and are class-independent by design; turning it
into a full pre-flight validator that resolves+adds the component and reports
its handle would duplicate the already-shipped validate path in
`blueprint.scs.add_component` (`FSCSHandlers::AddSCSComponent` →
`ResolveClassByName` + "Component class not found" / "Class is not a
component"), which is the path callers should reach for to pre-validate a
class. So the defect is narrow: the documented `componentClass` param is
accepted and echoed back as if it were probed even when it names a class that
does not exist or is not a `UActorComponent`, giving an identical
`success:true` for a typo'd/abstract class as for a real one.

`BlueprintCreationHandler.cpp` (`blueprint.probe_subobject_handle`, ~lines
382–499):

```cpp
FString ComponentClass = Ctx.GetString(TEXT("componentClass"), TEXT("StaticMeshComponent"));
// ... creates an EMPTY temp Blueprint at /Game/Temp/MCPProbe ...
ResultObj->SetStringField(TEXT("componentClass"), ComponentClass);   // echoed, never used
// ... GatherSubobjectDataForBlueprint(CreatedBP, GatheredHandles) on the EMPTY bp ...
ResultObj->SetBoolField(TEXT("success"), true);
```

The gathered handles come from the empty probe Blueprint's default
subobjects (root + scene root), which have **nothing to do** with
`componentClass`. There is no `FindObject<UClass>` / `LoadClass` /
class-existence check anywhere in the handler, so the requested class never
influences the outcome. Every call returns `success:true`.

**Why it matters:**

- The param is documented ("Component class to probe") and echoed back as if
  it were probed, but a caller pre-validating `StaticMeshComponent` gets the
  same `success:true` it would get for
  `ThisClassDoesNotExist_TotallyBogus_12345` or a non-component class. The
  result for a valid class and a bogus class are identical except the echoed
  `componentClass` field, so there is no JSON-level signal distinguishing
  "valid class" from "garbage".

This is the same class of footgun already tracked for other methods
(`B-material-stub-handlers-silent-success`,
`B-configure-world-partition-silent-noop`): a handler that echoes its input
back as if it were a meaningful result while doing none of the implied
validation. The board's house resolution for this family is to fail loud,
not to build a new capability.

**Fix (honesty fix only — do NOT build a second validator):**

Resolve `componentClass` to a `UClass` via the existing `ResolveClassByName`
(`Utils/ClassUtils.h`) and `SendError` instead of `success:true` when it
fails: `CLASS_NOT_FOUND` if the string does not resolve, and
`CLASS_NOT_A_COMPONENT` if it resolves but is not a `UActorComponent`
subclass. A `componentClass` that does resolve to a component still returns
the existing subsystem/SCS smoke-probe result unchanged. Do **not** add the
component to the temp Blueprint or report a per-class handle — that pre-flight
"resolve + add + report handle" capability already exists in
`blueprint.scs.add_component` and re-implementing it here would be
gold-plating a throwaway-BP smoke probe.

## Repro

1. `blueprint.probe_subobject_handle {componentClass:"StaticMeshComponent"}`
   → `{"componentClass":"StaticMeshComponent","success":true,"subsystemAvailable":true,"gatheredHandles":["SubobjectDataHandle@0x…","SubobjectDataHandle@0x…"],"message":"Native probe completed"}`
2. `blueprint.probe_subobject_handle {componentClass:"ThisClassDoesNotExist_TotallyBogus_12345"}`
   → `{"componentClass":"ThisClassDoesNotExist_TotallyBogus_12345","success":true,"subsystemAvailable":true,"gatheredHandles":["SubobjectDataHandle@0x…","SubobjectDataHandle@0x…"],"message":"Native probe completed"}`
   — identical `success:true` shape; the bogus class is accepted exactly like the valid one.
3. `blueprint.probe_subobject_handle {componentClass:"ActorComponent"}` (abstract-ish / non-scene base)
   → `success:true` as well. The result (2 gathered handles from the empty probe BP) is the same regardless of class, proving `componentClass` never influences the probe.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed via `mcp__editor-automation__call`. `blueprint.probe_subobject_handle` returns `success:true` for `StaticMeshComponent`, `RotatingMovementComponent`, `ActorComponent`, and a fabricated nonexistent class name alike, with identical result shape (2 `gatheredHandles` from an empty probe Blueprint). Confirmed in `BlueprintCreationHandler.cpp` (~382–499) that `componentClass` is read into a local and only echoed into the result — never resolved to a `UClass`, validated, or used; the gathered handles come from the empty temp BP's default subobjects, independent of the requested class. Silent success-with-no-effect: the documented "Component class to probe" param is decorative, so the method cannot serve its obvious purpose (pre-flight class validation). Dedup: ripgrep + board scan found no existing ticket for this method (the silent-success siblings `B-material-stub-handlers-silent-success` and `B-configure-world-partition-silent-noop` cover different methods).
- `#2-reword-and-honesty-fix` `IN-REVIEW` developer — Reworded to the minimum honesty fix (adversarial lens: original Option 1 "resolve+add+report handle" gold-plates the already-shipped `blueprint.scs.add_component` validate path; dropped severity High→Medium, retitled, collapsed two fix options into the fail-loud one). Implemented in `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BlueprintCreationHandler.cpp`: right after reading `componentClass`, resolve it via the existing `ResolveClassByName` (`Utils/ClassUtils.h`, already included) and `Ctx.SendError` before creating the probe BP — `CLASS_NOT_FOUND` when it does not resolve, `CLASS_NOT_A_COMPONENT` when it resolves but is not a `UActorComponent` subclass. A resolving component class still returns the unchanged subsystem/SCS smoke-probe result; no component is added to the temp BP (that capability stays in `blueprint.scs.add_component`). Regression tests in `Source/EditorAutomationRpcGateway/Private/Tests/Blueprint/TestBlueprintHandlers.cpp` (`blueprint.probe_subobject_handle.BogusClassErrors` asserts a fabricated class → `bSuccess=false`, `ErrorCode=CLASS_NOT_FOUND`; `.NonComponentClassErrors` asserts `Object` → `CLASS_NOT_A_COMPONENT`) via `InvokeHandlerWithCapture` — both flip back to `success:true` if the guard is reverted. Did not compile/run (later phase).
