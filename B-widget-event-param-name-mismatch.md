---
id: B-widget-event-param-name-mismatch
title: "`widget_event` entry parameter names silently require the UE delegate's canonical name"
status: DONE
severity: Medium
category: bug
tags: [bpir, widget-event, param-resolution]
---

# `widget_event` entry parameter names silently require the UE delegate's canonical name

`entry widget_event WidgetName.OnFoo(Type MyName)` lets you declare *any* identifier for the event parameter, but the compiler only resolves references to it (`$MyName` or bare `MyName`) when the declared name matches the UE delegate's internal parameter name exactly. Renaming the param — natural for readable BPIR — silently produces a "variable not found" error.

**Repro (observed this session, `/App/App/UI/ReplayEditor/W_ClipSettings_Pilot`):**

```
entry widget_event HudCheck.OnCheckStateChanged(enum<ECheckBoxState> NewCheckState) {
    %bChecked = call EqualEqual_ByteByte(A: $NewCheckState, B: 1)
    call SetHud(bNewHud: %bChecked)
}
```

Fails with:
```
COMPILE_FAILED: Line 2: Could not resolve value '$NewCheckState' for pin 'A'
 — variable 'NewCheckState' not found on W_ClipSettings_Pilot_C
 (for widgets: check 'Is Variable' in Designer)
```

Bare form `A: NewCheckState` also fails with the same message.

Changing the declared name to `bIsChecked` (the internal parameter name on UCheckBox's `FOnCheckBoxComponentStateChanged` delegate) compiles cleanly. The compiler is accepting the declared alias at parse time but never binding it to the actual event-parameter pin underneath — so only the canonical name works.

**Impact:**
- Forces authors to know (or decompile an existing example of) every UE delegate's internal parameter name to wire any non-trivial widget event.
- The error message misleads: "check 'Is Variable' in Designer" points at a widget-variable resolver path that has nothing to do with an event parameter.
- Round-trip from a BPIR file using a custom param name fails silently at first compile.

**Workaround:** Look up the UE delegate signature for each widget event and use its exact internal param name. Common examples:
- `UCheckBox.OnCheckStateChanged` → `bIsChecked` (despite being an `ECheckBoxState`)
- `UButton.OnClicked` → no params
- `USpinBox.OnValueChanged` → `InValue`

**Proposal:** When parsing a `widget_event` entry, register the declared param name as an alias for the real delegate-pin name in the PinResolver's lookup table, the same way function-entry params are bound. Fix the error message's "check 'Is Variable' in Designer" branch so it only fires for unresolved identifiers that aren't event parameters — for event params, say "param 'X' in entry declaration does not correspond to a pin on the event delegate; available params: ...".

## History
- `#1-canonical-param-name-required` `OPEN` reporter — Hit while wiring the Pilot / Static / Free / Path clip-settings widgets for the replay editor. First attempt used `NewCheckState` (matches the enum type). Compile failed with the widget-variable hint. Retry with `bIsChecked` succeeded. Lost ~2 minutes per widget on the same mistake across four widgets before internalising the rule. Similar impact likely every time an author declares new widget events with non-canonical param names.
- `#2-alias-registration-added` `IN-REVIEW` developer — `SetupWidgetEvent` (`BpirCompiler.cpp`) now accepts `Block.Params` and registers author-declared param names as aliases on the delegate output pins (positional match), in addition to keeping the canonical delegate pin names addressable. `case EBpirEntryKind::WidgetEvent:` dispatch updated to pass `Block.Params`. Header `BpirCompiler.h` updated. Covered by `FBpirWidgetEventParamAliasTest` in `Tests/Private/Bpir/TestWidgetEventParams.cpp`. Parallel limitation in `SetupComponentEvent` intentionally left as a separate future ticket.
- `#3-returned-still-fails-live` `OPEN` tester — Returned: live MCP repro still fails exactly per the original report. On `/Game/App/UI/Test/W_McpVerifyTemp` with a `HudCheck` CheckBox widget added, `compile_bpir` body `entry widget_event HudCheck.OnCheckStateChanged(enum<ECheckBoxState> MyCustomName) { %b = call EqualEqual_ByteByte(A: $MyCustomName, B: 1) }` → error `COMPILE_FAILED: Line 2: Could not resolve value '$MyCustomName' for pin 'A' — variable 'MyCustomName' not found on W_McpVerifyTemp_C (for widgets: check 'Is Variable' in Designer)`. Counter-test with canonical name: same body with `bIsChecked` as both the declared param name and the `$bIsChecked` reference → `compiled: true, nodeCount: 2`. The custom-name-as-alias behavior the fix advertised is not in effect — the PinResolver still only finds the canonical delegate pin name, and also still points at the misleading "check 'Is Variable' in Designer" hint instead of the event-param-specific guidance the fix proposed. `FBpirWidgetEventParamAliasTest` may be passing on isolated parser state but the live handler path through `compile_bpir` on a real widget doesn't route `Block.Params` into the resolver's alias table.
- `#4-phase2-alias-reapplied` `IN-REVIEW` developer — Extracted Phase-1 alias registration into `RegisterEntryParamAliases` in `BpirCompiler.cpp` and re-apply it after `PinResolver->Clear()` in Phase 2 (~line 1258) for WidgetEvent/ComponentEvent/CustomEvent kinds; aliases now survive into `PreEmitVariableRefs`. Improved the `BpirCompiler.cpp:4823` error message to list available event-param pins when the unresolved identifier was declared on the current entry block. Pinned by `FBpirWidgetEventParamAliasLiveTest` in `TestWidgetEventParamAliasLive.cpp`. Counterfactual: if the Phase-2 re-registration is reverted, the assertion fails because `PinResolver->Clear()` wipes the Phase-1 aliases and `PreEmitVariableRefs` cannot resolve `$MyCustomName`.
- `#5-verified-alias-resolution` `DONE` tester — Verified live on `/Game/App/UI/Test/W_McpVerifyTemp` (with a `HudCheck` CheckBox added). `compile_bpir` body `entry widget_event HudCheck.OnCheckStateChanged(enum<ECheckBoxState> MyCustomName) { %b = call EqualEqual_ByteByte(A: $MyCustomName, B: 1) }` returned `success: true, nodeCount: 2, status: "UpToDate", errors: [], warnings: []`. Custom param name `MyCustomName` (not the canonical `bIsChecked`) was resolved by the Phase-2 re-registered alias path. Pre-fix this exact form returned `COMPILE_FAILED: Could not resolve value '$MyCustomName' for pin 'A' — variable 'MyCustomName' not found ...`.
