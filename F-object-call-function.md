---
id: F-object-call-function
title: "Generic object and component function invocation with parameters"
status: DONE
severity: High
category: feature
tags: [rpc, reflection, function-call, generic-primitive]
---

# Generic object and component function invocation with parameters

`actor.call_function` is too narrow to serve as a safe generic primitive. It only targets placed actors and calls `ProcessEvent` on the actor. When the target `UFunction` has parameters, the current call shape allocates a zeroed parameter buffer and cannot pass caller-provided values or return out parameters/results.

This blocks hard-removal of shortcut RPCs whose only unique behavior is "call this reflected function with real arguments" or "call this component/object function", such as sky-light recapture, actor functions with scalar inputs, and Niagara component lifecycle operations.

**Workaround:** Keep the specialized RPCs where real parameters, return values, components, or arbitrary `UObject` targets are required. Use `actor.call_function` only for parameterless actor functions.
**Proposal:** Add a canonical `object.call_function`-style primitive that can target actors, components, assets, and arbitrary object paths; convert JSON arguments through the same property-conversion layer used by `property.set`; serialize return values and out params; and reject unsupported parameter types explicitly instead of calling with zeroed/default values.

## History
- `#1-hard-removal-gap` `OPEN` reporter — During the hard-removal audit, `actor.call_function` was rejected as a replacement for parameterized actor calls and component/object calls because it cannot pass arguments, cannot return values, and only targets actors.
- `#2-add-object-call-function` `IN-REVIEW` developer — Added `object.call_function` generic primitive at `Private/Handlers/Reflection/ObjectCallFunctionHandler.cpp` with full FProperty parm-buffer handling (InitializeValue/DestroyValue lifecycle, CPF_ReturnParm/OutParm/ConstParm/ReferenceParm classification), JSON→FProperty via PropertyUtils::ApplyJsonValueToProperty, return + out param serialization via ExportPropertyToJsonValue. New helper `AssetUtils::ResolveUObjectByPath`. Regression tests under `Tests/Private/Reflection/`.
- `#3-crash-during-object-call` `OPEN` tester — Crashed: live verification spawned `/Game/System/FrontEnd/Maps/L_Core.L_Core:PersistentLevel.StaticMeshActor_1`, `object.call_function` reported success for `SetActorScale3D` with `NewScale3D`, but `actor.get_transform` still returned scale `[1,1,1]` and `object.call_function` `GetActorScale3D` returned `[0,0,0]`; cleanup via `actor.delete` then failed with `fetch failed`, and `curl http://127.0.0.1:19880/health` could not connect.
- `#4-returned-scale-not-applied` `OPEN` tester — Returned: `object.call_function` reported void success for `SetActorScale3D` with `NewScale3D={x:2,y:3,z:4}`, but `actor.get_transform` still returned scale `[1,1,1]` and `object.call_function` `GetActorScale3D` returned `[0,0,0]`; cleanup deleted `McpVerify_ObjectCallFunction` and backend health stayed ready. Test: `actor.spawn` StaticMeshActor, `object.call_function` SetActorScale3D/GetActorScale3D, `actor.get_transform`, `actor.delete`.
- `#5-aligned-param-buffer` `IN-REVIEW` developer — Replaced the `object.call_function` hand-built `TArray<uint8>` ProcessEvent frame with an aligned `UFunction::ParmsSize` buffer using `UFunction::GetMinAlignment()`, initialized/destroyed only `CPF_Parm` slots while preserving `PropertyUtils::ApplyJsonValueToProperty` and `ExportPropertyToJsonValue`, and added regression coverage for `SetActorScale3D`/`GetActorScale3D` vector parameters in `TestObjectCallFunctionHandler.cpp`.
- `#6-verify-scale-applied` `DONE` tester — Verified: spawned StaticMeshActor `McpVerify_F_ObjectCallFunction`, `object.call_function` `SetActorScale3D` with `NewScale3D={x:2,y:3,z:4}` returned `void:true`, `object.call_function` `GetActorScale3D` returned `[2,3,4]`, cleanup via `actor.delete` succeeded with `existsAfter:false`.
