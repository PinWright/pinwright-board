---
id: E-bpir-set-on-createwidget-needs-cast
title: "BPIR `set %ref.Prop = value` rejects K2Node_CreateWidget output; requires explicit cast"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, set, create-widget, property-access, ergonomics]
---

# BPIR `set %ref.Prop = value` rejects K2Node_CreateWidget output; requires explicit cast

When the target of a BPIR `set` is the output of a `K2Node_CreateWidget` call, the compiler errors with `Could not find target pin '' on node 'Set'`. The node's return pin type *should* be known at compile time (it is typed to the `Class:` parameter), so property access should resolve directly — but it doesn't. The workaround is a redundant `cast<W_Class>(%item)` step before the `set`, which also forces the body into a separate block label.

## Repro

Fails:

```
%item = call K2Node_CreateWidget(Class: /App/.../W_MyReplayListItem.W_MyReplayListItem_C, OwningPlayer: %player)
set %item.Replay = %loop.ArrayElement
set %item.ParentScreen = self
call AddChild(Target: $TrackItemsContainer, Content: %item)
```

Error:
```
COMPILE_FAILED: Line 3: Could not find target pin '' on node 'Set'; Line 4: Could not find target pin '' on node 'Set'
```

Works (with awkward cast + block split):

```
%item = call K2Node_CreateWidget(Class: /App/.../W_MyReplayListItem.W_MyReplayListItem_C, OwningPlayer: %player)
%cast = cast<W_MyReplayListItem_C>(%item) [success -> @setup]

@setup:
    set %cast.AsWMyReplayListItem.Replay = %loop.ArrayElement
    set %cast.AsWMyReplayListItem.ParentScreen = self
    call AddChild(Target: $TrackItemsContainer, Content: %cast.AsWMyReplayListItem)
```

## Impact

Low. The workaround is two extra lines. But every "create a typed widget, assign a couple of instance-editable properties, add to container" flow — a very common list-rebuild pattern — is forced to split into two blocks and introduce a cast that conveys no information beyond what the `Class:` parameter already stated.

The same issue applies to any K2Node-flavored call whose return pin is typed via a class parameter (e.g. potentially `K2Node_SpawnActorFromClass`). Worth verifying as part of the fix.

## Proposed fix

Prefer `UK2Node_ConstructObjectFromClass::GetClassToSpawn()` over the pin's `PinSubCategoryObject` when resolving a `%ref` target class. `GetClassToSpawn()` reads the authoritative `ClassPin->DefaultObject` set by `CreateExpandNode`, and stays correct even when `ReconstructNode` doesn't fully propagate subcategory specialization. Applies uniformly to CreateWidget, SpawnActor, and any other `ConstructObjectFromClass` subclass. Implemented as a public static `FBpirCompiler::GetAuthoritativePinClass` helper so the behavior can be unit-tested independently of full BPIR compile.

## History
- `#1-set-after-createwidget-fails` `OPEN` reporter — Hit during replay subtask #4 W_MyReplaySelect rebuild. Needed to populate `Replay` and `ParentScreen` on each spawned list item after `K2Node_CreateWidget`; had to introduce a cast + separate block even though the widget class was fully known from the Class parameter. Working BPIR is now live in `/App/App/UI/LobbyAndMenu/W_MyReplaySelect.RebuildList`.
- `#2-get-class-to-spawn-preferred` `IN-REVIEW` developer — Reshaped from "pin-type inspection logic via cast<T>" to preferring `UK2Node_ConstructObjectFromClass::GetClassToSpawn()` in `FBpirCompiler::ResolveTargetClass`. New public static `GetAuthoritativePinClass` helper in BpirCompiler.h/.cpp covers all ConstructObjectFromClass subclasses (CreateWidget, SpawnActor, etc.). Pinned by FBpirResolveTargetClassConstructObjectTest in TestBpirResolveTargetClass.cpp.
- `#3-returned-still-fails-set-ref` `OPEN` tester — Returned: the set-through-%ref-of-CreateWidget path still fails identically to the original ticket. Repro on a fresh widget with a variable: created `/Game/App/UI/Test/W_McpVerifyTemp` (WidgetBlueprint), added int variable `TestInt`. Call 1 — failing direct form: `compile_bpir` body `%p = call GetOwningPlayer(Target: self); %item = call K2Node_CreateWidget(Class: /Game/App/UI/Test/W_McpVerifyTemp.W_McpVerifyTemp_C, OwningPlayer: %p); set %item.TestInt = 42` → error `COMPILE_FAILED: Line 4: Could not find target pin '' on node 'Set'` (identical empty-target-pin signature from the original report). Call 2 — same with short class name `Class: W_McpVerifyTemp_C` → same error. Call 3 — workaround form with explicit cast: `%cast = cast<W_McpVerifyTemp_C>(%item) [success -> @ok, fail -> @done]; @ok: set %cast.AsWMcpVerifyTemp.TestInt = 99; @done:` → `compiled: true, nodeCount: 5`. Conclusion: `GetAuthoritativePinClass` / `GetClassToSpawn()` is not flowing through to the `set`-instruction's property resolver for `%ref`-of-`K2Node_CreateWidget` — the cast workaround is still required. Either the helper isn't called from the `set` emission site, or `%item`'s recorded type is still `UserWidget*`/generic rather than the specialized `W_McpVerifyTemp_C`.
- `#4-generic-k2node-sets-default-object` `IN-REVIEW` developer — `CreateGenericK2Node` (CodeNodeEmitter.cpp) now sets `ClassPin->DefaultObject` on any `UK2Node_ConstructObjectFromClass` node by resolving the call's `Class:` arg before `ReconstructNode`, so `GetClassToSpawn()` returns the real spawn class for `call K2Node_CreateWidget(Class: ...)` forms that miss the short-name registry. Added a defensive fallback in `FBpirCompiler::GetAuthoritativePinClass` to resolve `ClassPin->DefaultValue` as a class path when `GetClassToSpawn()` returns null. Pinned by `FBpirSetAfterCreateWidgetTest` in `TestBpirSetAfterCreateWidget.cpp`. Counterfactual: reverting the `DefaultObject` assignment in `CreateGenericK2Node` makes the test fail with `Could not find target pin '' on node 'Set'`.
- `#5-verified-direct-set` `DONE` tester — Verified live on `/Game/App/UI/Test/W_McpVerifyTemp` (TestInt int variable added). `compile_bpir` body `entry custom_event T_setcw() { %p = call GetOwningPlayer(Target: self); %item = call K2Node_CreateWidget(Class: /Game/App/UI/Test/W_McpVerifyTemp.W_McpVerifyTemp_C, OwningPlayer: %p); set %item.TestInt = 42 }` returned `success: true, nodeCount: 5, status: "UpToDate", errors: [], warnings: []`. No cast or block split required; `set %item.TestInt` resolved directly off the CreateWidget output. Pre-fix returned `COMPILE_FAILED: Could not find target pin '' on node 'Set'`.
