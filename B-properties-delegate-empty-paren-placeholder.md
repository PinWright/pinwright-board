---
id: B-properties-delegate-empty-paren-placeholder
title: "Delegate properties emit identical '()' placeholder regardless of binding state"
status: DONE
severity: Low
category: bug
tags: [properties, delegates]
---

# Delegate properties emit identical '()' placeholder regardless of binding state

Both `FMulticastInlineDelegateProperty` and `FMulticastSparseDelegateProperty` serialize to `"value": "()"` with no binding metadata, regardless of whether the delegate is bound to nothing, one function, or many. The `_kind` field distinguishes the delegate type but the value carries no signal about whether bindings exist.

Sample paths:
- `App/App/UI/.../W_PauseMenu/properties.json` (`OnCalibrateClicked` as `FMulticastInlineDelegateProperty`, value `"()"`)
- `App/App/UI/.../B_UIArrow/properties.json` (similar with `FMulticastSparseDelegateProperty`)

## Fix sketch

In `PropertyUtils.cpp`'s delegate-property serializer, enumerate `FMulticastScriptDelegate::GetAllObjects()` (or the equivalent for sparse delegates) and emit:

```json
"OnCalibrateClicked": {
    "_kind": "FMulticastInlineDelegateProperty",
    "type": "...DynamicMulticastDelegate",
    "bindings": [
        { "object": "...", "function": "OnCalibrateClickedHandler" }
    ]
}
```

Or, if bindings are intentionally not surfaced in the dump (because they're runtime/instance state), document the `"()"` as canonical "unbound or runtime-resolved" sentinel.

## History
- `#2-typed-delegate-bindings` `IN-REVIEW` implementer — delegate properties now emit typed objects with `bindings` and `bindingStatus`; empty delegates no longer use raw `"()"` as the success shape. Added utility coverage for empty inline and sparse delegates.
- `#1-delegate-placeholder` `OPEN` reporter — both delegate kinds collapse to `"()"`; cannot tell from dump whether a delegate has bindings or not.
- `#3-verify-fix` `DONE` tester — Verified: fresh `asset.dump` on `/App/App/UI/LobbyAndMenu/TrackMenu/W_PauseMenu` and `/App/App/UI/B_UIArrow`; `OnCalibrateClicked` (inline) and all sparse delegates (e.g. `OnBeginCursorOver`, `OnClicked`, `PhysicsVolumeChangedDelegate`) now emit `{ "_kind": "...DelegateProperty", "bindings": [], "bindingStatus": "empty", "type": "<sig>" }`; no raw `"()"` placeholder present.
