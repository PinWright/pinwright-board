---
id: B-array-get-item-connection-fail
title: "BPIR `Array_Get` result can't be connected to subsequent `cast<>`"
status: DONE
severity: Medium
category: bug
tags: []
---

# BPIR `Array_Get` result can't be connected to subsequent `cast<>`

BPIR `call Array_Get(TargetArray: ..., Index: 0)` produces a result that cannot be passed into a subsequent `cast<>` instruction. Using `.Item` accessor errors with `TryCreateConnection failed wiring data 'Item' -> 'Object'`.

**Repro:**
```
entry widget_event SaveButton.OnClicked() {
    %r = call GetAllWidgetsOfClass(WorldContextObject: self, WidgetClass: ..., TopLevelOnly: false)
    %h = call Array_Get(TargetArray: %r.FoundWidgets, Index: 0)
    %c = cast<W_ReplaySaveHandler_C>(%h.Item) [success -> @ok]
@ok:
    ...
}
```
→ `COMPILE_FAILED: Line 4: TryCreateConnection failed wiring data 'Item' -> 'Object'`

**Workaround:** Iterate via `foreach` and cast `%loop.ArrayElement` instead:
```
%loop = foreach(%r.FoundWidgets) [body -> @body, completed -> @done]
@body:
    %c = cast<W_ReplaySaveHandler_C>(%loop.ArrayElement) [success -> @ok]
    ...
```

The `.ArrayElement` accessor works correctly; `.Item` does not. Either `.Item` is the wrong field name for `Array_Get`'s wildcard out-param, or the typing resolution fails at connection time.

**Fix:** Document the correct accessor for Array_Get's result, OR make `.Item` work (resolve wildcard to connection target's inferred type).

## History
- `#1-initial-repro` `OPEN` reporter — Hit while implementing cross-widget "find handler by class" logic. `foreach` workaround succeeded but adds a loop for what should be a single lookup.
- `#2-two-part-pin-fix` `IN-REVIEW` developer — Two-part fix. (1) `BpirValueResolver::FindOutputPinByName` (lines 664-716) adds a third fallback pass: when `PinName` is one of `"Item"`/`"Result"`/`"Value"` (case-insensitive) AND the node has exactly one non-exec output pin, returns that pin — maps BPIR `.Item` accessor onto K2Node_CallArrayFunction's `"Output"` wildcard pin. (2) `BpirCompiler::WireDataPins` (lines 4101-4119) now calls `Schema->NotifyPinConnectionListChanged(TargetPin)` after successfully wiring a pin named `"TargetArray"` — triggers UE's wildcard-to-element-type propagation on `UK2Node_CallArrayFunction` / `UK2Node_GetArrayItem` so downstream `cast<>` connections see the resolved element type. Scoped by pin-name to avoid firing on every wire; no new headers needed.
- `#3-returned-fix-insufficient` `OPEN` tester — Returned: above fix was insufficient. Live test `call Array_Get(...)` + `cast<T>(%h.Item)` still failed with `TryCreateConnection failed wiring data 'Item' -> 'Object'`. Root cause: BPIR's `CodeNodeEmitter::CreateCallFunctionNode` instantiates generic `UK2Node_CallFunction` for ALL call instructions, including Array_Get. The base class's `NotifyPinConnectionListChanged` has no wildcard propagation — only `UK2Node_CallArrayFunction::NotifyPinConnectionListChanged` (via `PropagateArrayTypeInfo`) does. Schema notification was firing on a node that didn't know how to propagate.
- `#4-call-array-function-node` `IN-REVIEW` developer — Real fix in `CodeNodeEmitter.cpp::CreateCallFunctionNode`: now instantiates `UK2Node_CallArrayFunction` when the UFunction has `ArrayParm` metadata (Array_Get, Array_Add, Array_Contains, etc.); falls back to generic `UK2Node_CallFunction` otherwise. With the correct K2Node class, UE's schema `TryCreateConnection` → `MakeLinkTo` → `PinConnectionListChanged` chain fires `PropagateArrayTypeInfo` automatically. Refactor pass removed the now-redundant explicit `NotifyPinConnectionListChanged` block in `BpirCompiler::WireDataPins`. Added automation test `FCompilerIntegrationArrayGetCastTest`.
- `#5-verified-array-get-cast` `DONE` tester — Verified via MCP: `blueprint_compile_bpir` on `W_McpVerifyTemp` with body `%r = call GetAllWidgetsOfClass(...); %h = call Array_Get(TargetArray: %r.FoundWidgets, Index: 0); %c = cast</Script/App.ReplaySaveHandlerWidget>(%h.Item) [success -> @ok]; @ok: call PrintString(...)` returned `nodeCount:6, errors:[], compiled:true, status:"UpToDate"`. Before fix: same input failed with `TryCreateConnection failed wiring data 'Item' -> 'Object'`.
