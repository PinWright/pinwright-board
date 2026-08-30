---
id: B-bpir-override-param-not-resolvable
title: "BPIR override-entry params are declared but not registered in the param resolver"
status: DONE
severity: High
category: bug
tags: [bpir, override, entry, param-resolution, blueprint-implementable-event]
---

# BPIR override-entry params are declared but not registered in the param resolver

After `#2-register-all-event-output-pins` and `#3-const-ref-classified-as-input`, the remaining live repro narrowed to a smaller hole: **single-level `$Param.Field` access on struct-typed override/event params**. The param itself is now registered correctly; the failure was in the resolver's `$target.property` pre-emit path, which still assumed the target was object-typed.

An `entry override <Name>(T Param) { ... }` or `entry event <Name>(T Param) { ... }` for a `BlueprintImplementableEvent` or `BlueprintNativeEvent` accepts the param declaration at parse time — the emitted graph's entry node even round-trips through `blueprint.decompile` as `entry override <Name>(struct<T> Param)` — but references to `$Param` or bare `Param` in the body fail with `variable 'Param' not found on <BP>_C (for widgets: check 'Is Variable' in Designer)`. The param is declared in the entry signature but never registered with the param resolver so it can't be read.

## Repro (observed this session, `W_ClipSettings_Pilot`)

Parent C++:

```cpp
UFUNCTION(BlueprintImplementableEvent, Category = "Replay Editor|Clip Settings|Pilot", meta = (DisplayName = "On Clip Loaded"))
void BP_OnClipLoaded(const FEdlPilotPayload& InPayload);
```

BPIR that compiles (no body references to `$InPayload`):

```
entry event BP_OnClipLoaded(const struct<FEdlPilotPayload>& InPayload) {
    call PrintString(InString: "Clip loaded", bPrintToLog: true, bPrintToScreen: false)
}
```

→ Success. `blueprint.decompile` emits `entry override BP_OnClipLoaded(struct<EdlPilotPayload> InPayload)` — round-trips as an override, param present in the signature.

BPIR that fails (any body reference to `$InPayload`):

```
entry event BP_OnClipLoaded(const struct<FEdlPilotPayload>& InPayload) {
    call PrintString(InString: $InPayload.PilotUid, bPrintToLog: true, bPrintToScreen: false)
}
```

→ `COMPILE_FAILED: Line 2: Could not resolve value '$InPayload.PilotUid' for pin 'InString' — variable 'InPayload' not found on W_ClipSettings_Pilot_C (for widgets: check 'Is Variable' in Designer)`.

Bare form `InPayload.PilotUid` fails identically. `$InPayload` alone (no field access) also fails.

Additional data point: using `entry override` explicitly with the SAME signature errors with `Override 'BP_OnClipLoaded' does not match the parent signature: Input count mismatch: expected 1, actual 0` — BPIR found the parent function (with 1 param) but parsed the override's declared params as 0. So override-entry parameter parsing is broken *before* emit, while `entry event` (which falls through to the override path for BIE target names) parses them at declaration time but drops them before param-resolver registration.

## Why this matters

BlueprintImplementableEvent with struct params is the canonical hook pattern for BP subclasses to react to C++ state changes — e.g., C++ `LoadClip(payload)` calling BP_OnClipLoaded so the BP can push fields into widgets. Without working param resolution, the override is inert — it fires but the BP can't read the payload to update anything.

Custom events work (`entry custom_event MyHandler(const struct<T>& P) { call Foo(Bar: $P.Field) }` — the const-ref fix from `B-bpir-bind-dispatcher-external-target-local-event` round 3 made this case work). Override-entry params should follow the same resolver registration path but don't.

## Impact

Any BPIR-authored override of a BP_Implementable/BlueprintNative event with a struct, enum, or object parameter cannot read its own argument. Blocks every "C++ fires, BP reacts with argument" pattern — the most common cross-language extension point in UE.

Workaround: none that doesn't resort to editing the BP in the UE editor after BPIR or routing the data through a side-channel variable. Neither is an acceptable authoring workflow.

## Fix

`FBpirCompiler::SetupBuiltinEvent` (`BpirCompiler.cpp` ~line 2202) was special-casing only ~5 hardcoded engine event names (`ReceiveTick`, `ReceiveActorBegin/EndOverlap`, `ReceiveHit`, `ReceiveAnyDamage`, `ReceiveEndPlay`) and registering their canonical pins with `PinResolver`. For any other `BlueprintImplementableEvent` / `BlueprintNativeEvent` override authored as `entry event <Name>(...)`, `CreateEventNode` produced a `UK2Node_Event` whose `AllocateDefaultPins()` correctly populated the output pins from the parent UFUNCTION (so `InPayload` etc. existed on the node), but no `PinResolver` registration ran — leaving body references like `$InPayload` unresolvable.

Replaced the hardcoded if/else cascade with a generic loop that walks all output pins on the event node, skips Exec / Delegate / MCDelegate categories and `PN_Self` / `PN_Then`, and registers each remaining pin via `PinResolver->RegisterVariable(Pin->PinName.ToString(), Pin)`. This mirrors the loop already in `SetupOverride` (`BpirCompiler.cpp` ~line 2826) — keep the two in sync. Purely additive: every previously hardcoded event still gets its canonical params registered, plus arbitrary BIE/BNE overrides become first-class.

**Secondary half (`entry override` "Input count mismatch: expected 1, actual 0"):** root-caused to `BlueprintHandlerUtils::GetFunctionSignatureDescriptors` (and `BlueprintApiIndexHandler`) testing `CPF_OutParm` without filtering `CPF_ConstParm`. UE's `CPF_ReferenceParm` doc (in `ObjectMacros.h`) explicitly states that `CPF_OutParm` is set on every reference param — including `const T&`. UE's canonical classifier (`EdGraphSchema_K2.cpp:938`, `KismetCompiler.cpp:221`) is `CPF_OutParm && !CPF_ConstParm`. Without that filter, `BlueprintImplementableEvent void BP_OnClipLoaded(const FEdlPilotPayload& InPayload)` introspected as 0 inputs / 1 output — `InPayload` was misclassified into `OutOutputs`. Fix: add the missing const-param check to both classifiers.

**Tertiary half (`$struct.field` pre-emit cache):** after the two earlier fixes, the remaining widget repro was not a `SetupOverride` registration issue. `PreEmitVariableRefs()` still routed `$InPayload.PilotUid` through `FBpirValueResolver::PreEmitExternalGet`, and that method still hard-cast the target pin's `PinSubCategoryObject` to `UClass`. Struct-typed entry params therefore never populated `ExternalGetCache`, and `ResolveValue("$InPayload.PilotUid")` later failed even though the underlying event pin had already been registered with `PinResolver`.

Fix: branch on `PC_Struct` in `PreEmitExternalGet` before the existing object/class path. For struct targets, delegate to `ResolveStructMemberThroughPin(TargetPin, PropertyName)` and cache the returned member pin under the existing `Target.Property` key. This keeps the current `$target.property` contract intact while reusing the already-shipped auto-break/native-break machinery used by `%ref.StructMember` and chained `%ref.Pin.Prop` resolution.

**Related "subset-only" issues found, filed separately:**
- `B-bpir-component-event-params-not-registered` — `SetupComponentEvent` only registers 5 hardcoded overlap/hit pin names; every other component delegate's params are unresolvable.
- `B-bpir-widget-event-canonical-name-not-registered-without-param-decl` — `SetupWidgetEvent` gates the entire registration block on `Params.Num() > 0`, so canonical pin names (e.g. `bIsChecked`) become unresolvable when the author omits explicit param declarations.
- `E-bpir-dollar-chained-struct-field` — `$name.a.b` chained access still lacks parity with the `%ref` chained resolver and remains a separate follow-up.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while adding BP_OnClipLoaded overrides to four typed clip-settings widgets (W_ClipSettings_Pilot/Static/Free/Path). The parent C++ `UClipSettings_PilotWidget::BP_OnClipLoaded(const FEdlPilotPayload& InPayload)` is a clean `UFUNCTION(BlueprintImplementableEvent, ...)`. Verified UFUNCTION is live by compiling a body-less override successfully (round-trips through `blueprint.decompile` as `entry override BP_OnClipLoaded(struct<EdlPilotPayload> InPayload)`, and `blueprint.inspect` lists `BP_OnClipLoaded()` in the Events section). As soon as the body references `$InPayload` for anything — field access or straight pass-through — the `Could not resolve value` / `variable not found` error fires. Identical shape across all four intended overrides (Pilot/Static/Free/Path payload structs, all `BlueprintImplementableEvent` with a single `const <Payload>&` param). No existing board entry matches — `B-widget-event-param-name-mismatch` is narrower (widget delegate event param names not matching UE canonical names); this ticket is BPIR's param-resolver missing registration for override/BIE-event entries entirely, regardless of param name.
- `#2-register-all-event-output-pins` `IN-REVIEW` developer — Replaced the hardcoded if/else cascade in `FBpirCompiler::SetupBuiltinEvent` (`BpirCompiler.cpp` ~lines 2202-2254) with a generic loop that registers every output non-exec / non-delegate pin on the event node (skipping `PN_Self` / `PN_Then`), mirroring the existing loop in `SetupOverride`. Every BIE/BNE override authored via `entry event <Name>(...)` now gets its parent-UFUNCTION-derived params auto-registered with `PinResolver`. Regression test added at `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirOverrideEventParamResolution.cpp` (`FBpirOverrideEventParamResolutionTest`) — compiles `entry event ReceivePointDamage() { set StoredDamage = $Damage }` (a BIE NOT in the previously hardcoded list) on a transient AActor BP and asserts compile success. Counterfactual: with the loop reverted to the cascade, `ReceivePointDamage` matches none of the special cases, `$Damage` is never registered, and `set StoredDamage = $Damage` fails with "variable 'Damage' not found" → `Result.bSuccess == false`.
- `#3-const-ref-classified-as-input` `IN-REVIEW` developer — Fixed the secondary `entry override` "Input count mismatch: expected 1, actual 0" half. Added `&& !CPF_ConstParm` to the output-classification check in `BlueprintHandlerUtils::GetFunctionSignatureDescriptors` (`BlueprintHandlerUtils.cpp` line 551) and `BlueprintApiIndexHandler` (`BlueprintApiIndexHandler.cpp` line 103), matching UE's canonical pattern (`EdGraphSchema_K2.cpp:938`, `KismetCompiler.cpp:221`). Const-ref params (the BIE/BNE payload pattern) now correctly classify as inputs instead of being misrouted into `OutOutputs` because `CPF_OutParm` is implied by `CPF_ReferenceParm`. Regression test added at `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirOverrideConstRefSignature.cpp` (`FBpirOverrideConstRefSignatureClassificationTest`) — calls `GetFunctionSignatureDescriptors(AActor::ReceiveHit)` (whose `Hit` param is `const FHitResult&`) and asserts `Inputs.Num() == 5`, `Outputs.Num() == 0`, and `Hit` appears in `Inputs`. Counterfactual: with `&& !CPF_ConstParm` removed, `Hit` is misrouted to `OutOutputs`, dropping `Inputs.Num()` to 4 and pushing `Outputs.Num()` to 1 — both equality assertions fail. Two related "subset-only" patterns surfaced during this investigation are filed separately as `B-bpir-component-event-params-not-registered` and `B-bpir-widget-event-canonical-name-not-registered-without-param-decl`.
- `OPEN` tester — Returned: IN-REVIEW fix does not cover the live repro. After plugin rebuild + editor reconnect, retrying the exact shape from `#1-initial-repro` still fails:
  ```
  entry override BP_OnClipLoaded(const struct<FEdlPilotPayload>& InPayload) {
      %uidText = call Conv_StringToText(InString: $InPayload.PilotUid)
      call SetText(Target: $PilotLabel, InText: %uidText)
      %fovFloat = call Conv_IntToDouble(InInt: $InPayload.Fov)
      call SetValue(Target: $FovSpin, NewValue: %fovFloat)
      call SetIsChecked(Target: $HudCheck, InIsChecked: $InPayload.bHud)
  }
  ```
  on `/App/App/UI/ReplayEditor/W_ClipSettings_Pilot` (parent `UClipSettings_PilotWidget`, UUserWidget subclass). Errors:
  - `Line 2: Could not resolve value '$InPayload.PilotUid' for pin 'InString' — variable 'InPayload' not found on W_ClipSettings_Pilot_C (for widgets: check 'Is Variable' in Designer)`
  - `Line 4: Could not resolve value '$InPayload.Fov' for pin 'InInt' — variable 'InPayload' not found on W_ClipSettings_Pilot_C (for widgets: check 'Is Variable' in Designer)`
  - `Line 6: Could not resolve value '$InPayload.bHud' for pin 'InIsChecked' — variable 'InPayload' not found on W_ClipSettings_Pilot_C (for widgets: check 'Is Variable' in Designer)`
  - Cascaded `Unresolved function: 'SetText'` and `Unresolved function: 'Conv_IntToDouble'` (resolver collapses when their Target args can't resolve).
  The `#2-register-all-event-output-pins` fix added pin registration in `SetupBuiltinEvent` which covers `AActor`-parented `entry event ReceivePointDamage()` (its regression test). But BIE overrides on `UUserWidget` subclasses authored as `entry override <Name>(...)` apparently take a different code path — either `SetupOverride` (which the fix note claims was already correct) OR some third handler that still has the hardcoded-only registration. Hypothesis: the `entry override` path for BIE-on-widget routes through `SetupOverride`'s output-pin loop, but on a `UK2Node_Event` created with `FunctionReference.MemberName = BP_OnClipLoaded` and `FunctionReference.MemberParent = UClipSettings_PilotWidget` the pin enumeration yields zero non-exec output pins — the `AllocateDefaultPins` call may not be populating them because the BIE's UFUNCTION hasn't been fully introspected, or the pin iteration is skipping all pins due to a category mis-match (struct output pin vs the exec/delegate/self/then skip list). Suggest a third regression test using the exact live shape: UUserWidget subclass with `BIE void Fn(const FStruct& Param)`, compile `entry override Fn(const struct<FStruct>& Param) { ... $Param.Field ... }`, assert pin resolution succeeds. Without this widget-flavored test the fix keeps shipping without proving coverage on the authoring pattern the ticket opened against.
- `#5-verified-override-struct-field` `DONE` tester — Verified live on `/Game/App/UI/Test/BP_McpVerifyActor` (transient AActor BP). `compile_bpir` body `entry override ReceivePointDamage(float Damage, object<DamageType> DamageType, struct<Vector> HitLocation, struct<Vector> HitNormal, object<PrimitiveComponent> HitComponent, name BoneName, struct<Vector> ShotFromDirection, object<Controller> InstigatedBy, object<Actor> DamageCauser, const struct<HitResult>& HitInfo) { %s = call Conv_DoubleToString(InDouble: $HitLocation.X); call PrintString(InString: %s) }` returned `success: true, status: "UpToDate", errors: [], warnings: []`. Repeated with chained access `$HitInfo.Location.X` — also success. Pre-fix both forms returned `COMPILE_FAILED: Could not resolve value '$HitLocation.X' for pin 'InDouble' — variable 'HitLocation' not found ...`. Override-entry param registration + struct-typed `PreEmitExternalGet` branch both engaged.
- `#4-cache-struct-member-for-dollar-access` `IN-REVIEW` developer — Re-scanned the refactored resolver and confirmed the remaining failure was in `FBpirValueResolver::PreEmitExternalGet`, not in override setup. `$target.property` still depended on `ExternalGetCache`, but `PreEmitExternalGet` only handled object-typed targets via a hard `Cast<UClass>(PinSubCategoryObject)`. Struct-typed event/override params therefore never cached member pins for `$InPayload.Field`, even though the entry pin itself was already registered. Fix: add a `PC_Struct` branch before the object-class path, delegate to `ResolveStructMemberThroughPin(TargetPin, PropertyName)`, and cache the returned member pin under the same `Target.Property` key. Regression tests added at `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirDollarStructFieldAccess.cpp` (`FBpirDollarStructFieldAccessTest`) and `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirWidgetStructOverride.cpp` (`FBpirWidgetStructOverrideTest`). The actor test covers both native-break (`$HitLocation.X`) and `BreakStruct` (`$HitInfo.bBlockingHit`) shapes on `ReceivePointDamage`; the widget test uses a real `UWidgetBlueprint` overriding a test-only `BlueprintImplementableEvent void TestStructBIE(const FHitResult& InHit)` and verifies `$InHit.bBlockingHit` compiles and survives the post-BPIR full-compile/integrity pipeline. Filed mandatory follow-up `E-bpir-dollar-chained-struct-field` for still-unsupported `$name.a.b` chains.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 4 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
