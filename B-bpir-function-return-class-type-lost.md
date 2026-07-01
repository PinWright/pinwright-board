---
id: B-bpir-function-return-class-type-lost
title: "BPIR functions returning `object<T>` don't propagate T to downstream member-call resolution"
status: DONE
severity: Medium
category: bug
tags: [bpir, function, return-type, type-propagation, member-call]
---

# BPIR functions returning `object<T>` don't propagate T to downstream member-call resolution

When a BPIR `entry function` declares a return type of `object<SomeClass>` and callers store the result in a `%var`, later member-function calls through `Target: %var` fail with "Unresolved function" — even when the function genuinely exists on `SomeClass`. Inlining the cast that produces the typed reference resolves the issue, so the problem is specifically in how the compiler tags the caller's `%var` with the return class.

**Repro (observed this session, `/App/App/UI/ReplayEditor/W_ReplayEditor_ProgressModal`):**

```
entry widget_event CancelBtn.OnClicked() {
    %rootRef = call GetRootEditor(Target: self)
    %valid = call IsValid(Object: %rootRef)
    branch (%valid) [true -> @haveRoot, false -> @done]
@haveRoot:
    call CancelExport(Target: %rootRef)       # FAILS: Unresolved function 'CancelExport'
@done:
}

entry function GetRootEditor() -> object<AppReplayEditorWidget> {
    %found = call GetAllWidgetsOfClass(WorldContextObject: self, WidgetClass: ..., TopLevelOnly: false)
    %first = call Array_Get(TargetArray: %found.FoundWidgets, Index: 0)
    %typed = cast<W_AppReplayEditor_C>(%first.Item) [success -> @ok, fail -> @none]
@ok:
    return %typed
@none:
    return nullptr
}
```

Error:
```
Line 6: Unresolved function: 'CancelExport'. Searched: Blueprint class: W_ReplayEditor_ProgressModal_C, ... 360 more libraries
Line 3: Could not resolve value '%rootRef' for pin 'Object'
```

`CancelExport` is a `UFUNCTION` on `UAppReplayEditorWidget` (the C++ parent of `W_AppReplayEditor_C`), so with correct type propagation the resolver should find it.

**Workaround:** inline the cast at the call site; don't route the typed reference through a helper function.

```
entry widget_event CancelBtn.OnClicked() {
    %found = call GetAllWidgetsOfClass(...)
    %first = call Array_Get(TargetArray: %found.FoundWidgets, Index: 0)
    %typed = cast<W_AppReplayEditor_C>(%first.Item) [success -> @ok, fail -> @done]
@ok:
    call CancelExport(Target: %typed)         # WORKS
@done:
}
```

**Impact:** Forces per-call-site code duplication whenever an editor tree-of-widgets needs to reach a root, resolver subsystem, or any shared object obtained via a typed helper. Helper functions that return typed object references become unusable for their intended purpose.

**Proposal:** In the BPIR function-compilation path, make sure the return-value pin on `UK2Node_FunctionResult` carries the declared class in `PinType.PinSubCategoryObject`, and propagate that type to the caller's `%var` entry. The caller's variable should behave identically to a `%cast = cast<T>(...)` result for downstream resolution.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while wiring W_ReplayEditor_ProgressModal and W_ReplayEditor_ExportDialog to reach the root widget's `StartExport` / `CancelExport`. First attempt factored a `GetRootEditor()` helper that did the `GetAllWidgetsOfClass` + cast and returned `object<AppReplayEditorWidget>`. Caller's `call CancelExport(Target: %rootRef)` and `call StartExport(Target: %rootRef, ...)` failed with "Unresolved function". Rewrote both call sites to inline the cast — compiled cleanly. The same pattern of "helper function returning typed ref" failed in both widgets in the same way.
- `#2-return-type-tracking` `IN-REVIEW` developer — Targeted fix applied. `FBpirCompiler` now tracks BPIR-declared function return types in a `TMap<FName, FString> BpirDeclaredReturnTypes`, populated in `SetupFunction` and cleared per `Compile()` call. At each CallFunction emission site in `BpirCompiler.cpp` (~lines 3511 and 3570), the ReturnValue pin's `PinType` is patched from the declared type via `FCodePinResolver::ConvertCppTypeToPinType` when the callee is a BPIR-declared function in this compile batch. Covered by `FBpirFunctionReturnClassPropagatedTest` in `Tests/Private/Bpir/TestBpirFunctionReturnClass.cpp`. **Known residual risks (developer flag):** (a) does not handle chained helper calls through multiple hops — only the immediate CallFunction pin is patched; (b) if `UK2Node_CallFunction::ReconstructNode` runs after the patch due to downstream wiring, the patched pin type may be overwritten — mitigation places the patch after `SetFromFunction` and before further wiring; (c) cross-blueprint calls are out of scope. If the tester hits any residual case, return to `OPEN` with the specific failure.
- `#3-verified-type-propagation` `DONE` tester — Verified on `/Game/App/UI/Test/W_McpVerifyTemp` with a `HudCheck` CheckBox variable. `compile_bpir` body `entry custom_event DoTest() { %box = call GetCheckBox(Target: self); call SetIsChecked(Target: %box, InIsChecked: true) }` followed by `entry function GetCheckBox() -> object<CheckBox> { return $HudCheck }` → `compiled: true, nodeCount: 6`. The `SetIsChecked` call on `%box` resolved correctly — `SetIsChecked` is a UCheckBox member (not inherited by UUserWidget/UObject), so the caller had to know `%box` is specifically `UCheckBox*`. Earlier iteration mis-spelled the param as `bInIsChecked` and got the expected pin-name-hint error ("Did you mean 'InIsChecked'? Available pins: self, InIsChecked") — that error itself is proof the resolver enumerated SetIsChecked's pins, confirming type propagation to the call site succeeded. Without the fix, the failure would have been `Unresolved function: 'SetIsChecked'` as in the original report.
