---
id: B-bpir-asyncaction-decompile-factory-identity
title: "BPIR decompile loses async-action factory identity"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, async-action, round-trip]
---

# BPIR decompile loses async-action factory identity

`blueprint.decompile` currently emits configured `UK2Node_AsyncAction` nodes through the generic unknown-node fallback:

```bpir
%n1 = call K2Node_AsyncAction(OwningPlayer: %n0, WidgetClass: ..., LayerName: ..., bSuspendInputUntilComplete: true)
```

That output does not encode the actual async factory function, such as
`UAsyncAction_PushContentToLayerForPlayer::PushContentToLayerForPlayer`. Recompile only works when the compiler can infer the factory from the BPIR argument-name set. That inference is not an acceptable round-trip contract: argument names can collide, hidden/default pins can make the set non-unique, and new engine/plugin async actions can change the selected candidate.

## Repro

Decompile `/App/App/UI/W_AppMapEditor` from the cached dump:

```text
C:\Unity\unreal-fpv\.editor-automation\asset-dumps\App\App\UI\W_AppMapEditor\bpir.txt
```

Observed body:

```bpir
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) {
    %n0 = call GetOwningPlayer() @(316, 1902)
    %n1 = call K2Node_AsyncAction(OwningPlayer: %n0, WidgetClass: /App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu.W_HUD_DroneGameMenu_C, LayerName: (TagName="UI.Layer.Menu"), bSuspendInputUntilComplete: true) @(316, 1633)
}
```

Observed warning:

```text
Unknown node type (generic fallback): K2Node_AsyncAction
```

## Expected

Decompiler must preserve the configured async factory identity. Acceptable output should be explicit and deterministic, for example:

```bpir
%n1 = call K2Node_AsyncAction_PushContentToLayerForPlayer(...)
```

or another exact-factory BPIR form that resolves directly to the configured `UFunction`.

Compiler must then resolve that exact factory identity first. Anonymous `call K2Node_AsyncAction(...)` pin-set matching must be removed or changed to a hard error with a migration message; it must not silently select a factory by heuristic.

## Required Fix

1. Add a decompiler path for `UK2Node_AsyncAction` that reads the configured factory function from the node and emits exact factory identity instead of the raw node class name.
2. Add compiler support for the exact decompiled form if the current parser/resolver cannot already represent it reliably.
3. Remove the pin-set factory-selection fallback from `AutoConfigureAsyncTaskNode`; generic `K2Node_AsyncAction` without explicit factory identity should fail deterministically.
4. Update or replace the old pin-set matcher tests so success depends on exact factory identity, not argument-name inference.
5. Add a round-trip regression for `PushContentToLayerForPlayer`: decompile must emit factory identity, compile must recreate a `UK2Node_AsyncAction` configured for the same factory, and decompile again must preserve the same identity.

## History

- `#1-initial-report` `OPEN` reporter — Found in `W_AppMapEditor` BPIR dump. `K2Node_AsyncAction` decompiled through the generic fallback and relied on the compiler's `UBlueprintAsyncActionBase` argument-name superset matcher to become `PushContentToLayerForPlayer` again. This is unsafe; the fallback must not be treated as valid legacy behavior.
- `#2-exact-async-factory-identity` `IN-REVIEW` developer — Decompile configured async action nodes with exact factory identity, require exact `K2Node_AsyncAction_<Factory>` compilation instead of pin-set inference, added round-trip and generic-call rejection coverage, and updated BPIR async docs.
- `#3-verified-async-factory` `DONE` tester — Verified: ran `blueprint.decompile` on `/App/App/UI/W_AppMapEditor`; the MenuButton click handler now emits `call K2Node_AsyncAction_PushContentToLayerForPlayer(...)` with the exact factory suffix, no bare `K2Node_AsyncAction(` form appears anywhere, and the `warnings` array is empty (no generic-fallback warning).
