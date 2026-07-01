---
id: B-bpir-component-bound-event-decompile-identity
title: "BPIR decompile flattens widget-bound events to duplicate delegate signatures"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, widget-event, component-event, round-trip]
---

# BPIR decompile flattens widget-bound events to duplicate delegate signatures

`blueprint.decompile` currently emits `UK2Node_ComponentBoundEvent` nodes through the generic `UK2Node_Event` path:

```bpir
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) {
    ...
}
```

This loses the binding identity. Multiple widgets can legally bind to the same delegate signature, for example `MenuButton.OnClicked` and `StartButton_Race.OnClicked` both use `CommonButtonBaseClicked__DelegateSignature`. UE distinguishes these nodes by `UK2Node_ComponentBoundEvent::ComponentPropertyName`, `DelegatePropertyName`, and the generated `CustomFunctionName`; BPIR currently drops the component/widget name.

## Repro

Decompile `/App/App/UI/W_AppMapEditor` from:

```text
C:\Unity\unreal-fpv\.editor-automation\asset-dumps\App\App\UI\W_AppMapEditor\bpir.txt
```

Observed:

```bpir
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) {
    ...
}

entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) {
    ...
}
```

Those are not duplicate UE events by themselves; they are distinct widget-bound click handlers flattened into indistinguishable BPIR entries.

## Expected

Decompiler must emit the existing round-trippable entry kinds for component-bound events:

```bpir
entry widget_event StartButton_Race.OnClicked(object<CommonButtonBase> Button) {
    ...
}

entry widget_event MenuButton.OnClicked(object<CommonButtonBase> Button) {
    ...
}
```

For non-widget Blueprint component events, emit:

```bpir
entry component_event SomeComponent.SomeDelegate(...) {
    ...
}
```

## Required Fix

1. In decompiler entry-signature emission, detect `UK2Node_ComponentBoundEvent` before generic `UK2Node_Event`.
2. Emit `entry widget_event <ComponentPropertyName>.<DelegatePropertyName>(...)` for widget blueprints, and `entry component_event <ComponentPropertyName>.<DelegatePropertyName>(...)` for non-widget component-bound events.
3. Preserve delegate parameter pins exactly as today, but attach them to the specific widget/component event entry.
4. Add a regression with two widget buttons bound to the same `CommonButtonBaseClicked__DelegateSignature`; decompile must produce two distinct `entry widget_event ...` blocks, compile must recreate two distinct `UK2Node_ComponentBoundEvent` nodes keyed by `{ComponentPropertyName, DelegatePropertyName}`.
5. Add a negative or warning-path test proving duplicate plain `entry event <DelegateSignature>` blocks are not treated as a safe representation of component-bound events.

## History

- `#1-initial-report` `OPEN` reporter — Found in `W_AppMapEditor` BPIR dump. Two distinct `CommonButtonBase` button handlers decompile as duplicate `entry event CommonButtonBaseClicked__DelegateSignature(...)` blocks. This breaks round-trip because BPIR loses which widget owns each binding.
- `#2-component-bound-identity` `IN-REVIEW` developer — Decompile `UK2Node_ComponentBoundEvent` before generic events, emit widget/component event identity from Blueprint context, reject duplicate plain delegate-signature entries, and add regression coverage for distinct widget-bound delegates.
- `#3-verify-widget-event-identity` `DONE` tester — Verified: live decompile of `/App/App/UI/W_AppMapEditor` emits `entry widget_event StartButton_Race.OnButtonBaseClicked(...)` and `entry widget_event MenuButton.OnButtonBaseClicked(...)` as distinct entries; no duplicate plain `entry event CommonButtonBaseClicked__DelegateSignature` blocks remain.
