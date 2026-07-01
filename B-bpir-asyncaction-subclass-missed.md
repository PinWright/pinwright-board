---
id: B-bpir-asyncaction-subclass-missed
title: "BPIR pin-set matcher doesn't find `UAsyncAction_ShowConfirmation`"
status: DONE
severity: Medium
category: bug
tags: []
---

# BPIR pin-set matcher doesn't find `UAsyncAction_ShowConfirmation`

The `B-bpir-asyncaction-disambiguation` DONE fix routes `K2Node_AsyncAction_<anything>` through `UK2Node_AsyncAction::StaticClass()` and picks the best `UBlueprintAsyncActionBase` subclass by pin-name-set match. Works for `PushContentToLayerForPlayer` (tested in the DONE fix). **Does not work** for `UAsyncAction_ShowConfirmation::ShowConfirmationYesNo` — every form tried failed with `"Async Task: Missing Function"`, `"Could not find target pin 'InWorldContextObject'"`, etc.

**Attempts this session (all failed):**
- `call K2Node_AsyncAction(InWorldContextObject: self, Title: "...", Message: "...") [OnResult -> @r, then -> @t]`
- `call K2Node_AsyncAction_ShowConfirmationYesNo(...)`
- `call K2Node_AsyncAction_AsyncAction_ShowConfirmation(...)`
- `call K2Node_AsyncAction_ShowConfirmation(...)`
- `call ShowConfirmationYesNo(...)` — resolves to the factory function call, not the async-action K2Node; returns `UAsyncAction_ShowConfirmation*` proxy without `OnResult` exec or `Result` output pin
- `call AsyncAction_ShowConfirmation::ShowConfirmationYesNo(...)` — `Unresolved function`

`UAsyncAction_ShowConfirmation` IS registered (`system_inspect_inspect_class` returns it at `/Script/CommonGame.AsyncAction_ShowConfirmation`, parent `BlueprintAsyncActionBase`). The factory function signature matches what the `ShowConfirmationYesNo` K2Node expects: `static UAsyncAction_ShowConfirmation* ShowConfirmationYesNo(UObject* InWorldContextObject, FText Title, FText Message)`. The arg-name set BPIR provided (`InWorldContextObject, Title, Message`) is an exact match. Yet the pin-set matcher still emits "Missing Function".

**Impact this session:**
Replay list delete flow per design spec calls for `W_ConfirmationDialog` with "Удалить реплей?" before calling `RequestDelete`. Unable to wire via BPIR. Fell back to direct `RequestDelete` call — confirm-gate is missing.

**Workaround:**
Manually create the node via `mcp__editor_automation__.call path="blueprint.graph.create_node" args={"nodeType":"K2Node_AsyncAction"}` and wire pins by hand. Or leave the confirm-gate out of BPIR-authored delete flows.

**Possible root causes (one of these is likely):**
1. The `CommonGame` module's `UAsyncAction_ShowConfirmation` isn't in the subclass set the BPIR matcher iterates.
2. The pin-set matcher excludes subclasses whose factory function lives on the same class as the proxy (common Epic pattern: `static UAsyncAction_Foo* UAsyncAction_Foo::DoSomething(...)` rather than on a separate subsystem).
3. Some module-load-order issue — `CommonGame` subclass isn't registered by the time the matcher runs.

**Fix:**
Investigate which subclass enumeration path `AutoConfigureAsyncTaskNode` uses; verify `UAsyncAction_ShowConfirmation` is visited; add a regression test that pin-sets containing `{"InWorldContextObject", "Title", "Message"}` plus `ECommonMessagingResult` result type pick `UAsyncAction_ShowConfirmation`.

## History
- `#1-initial-repro` `OPEN` reporter — Hit during replay list delete-handler wiring. The same BPIR pattern that works for `PushContentToLayerForPlayer` does not work for `ShowConfirmationYesNo` despite `UAsyncAction_ShowConfirmation` being a registered subclass. Six syntax variants attempted, all failed with "Missing Function".
- `#2-named-factory-path` `IN-REVIEW` developer — Added named-factory resolution path at `BpirCompiler.cpp` call-resolution site (before the existing Pure/Impure split). New `FCodeNodeEmitter::IsAsyncActionFactory` helper applies UE's exact `BlueprintActionDatabaseRegistrar::IsFactoryMethod` rule (FUNC_Static, return type UBlueprintAsyncActionBase subclass, owner class not deprecated) plus the `HasDedicatedAsyncNode` opt-out on the owner class. New `FCodeNodeEmitter::CreateAsyncActionNode` spawns `UK2Node_AsyncAction`, calls `InitializeProxyFromFunction(Func)` BEFORE `AllocateDefaultPins` (order-critical per UE K2Node_AsyncAction.cpp:71-83), then wires exec in/out. This makes `call ShowConfirmationYesNo(...)` (and any other async factory by name) produce a correctly-configured async node with typed `Result` pin and delegate exec outputs. Pin-set matcher in `AutoConfigureAsyncTaskNode` stays untouched as the fallback for the anonymous `call K2Node_AsyncAction(...)` form.
- `#3-verified-on-mcp-replay` `DONE` tester — Verified on `W_McpVerify_Replay03`: `call ShowConfirmationYesNo(InWorldContextObject: self, Title: "Test Title", Message: "Test Message?") [OnResult -> @r, then -> @t]` compiled clean, produced a `K2Node_AsyncAction` with title `ShowConfirmationYesNo` and the full factory pin set — `execute`/`then`/`OnResult` exec pins, `Result` typed `byte` with `pinSubType: ECommonMessagingResult`, plus `InWorldContextObject` (object), `Title` (text, `defaultTextValue: "Test Title"`), `Message` (text, `defaultTextValue: "Test Message?"`). Six original failing syntaxes from the OPEN report would now work; the named-factory path is the canonical way to wire `ShowConfirmationYesNo` and other factory-style async actions from BPIR.
