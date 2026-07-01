---
id: B-bpir-asyncaction-dedicated-subclass
title: "BPIR named-factory path can't wire async actions with HasDedicatedAsyncNode (e.g. ListenForGameplayMessages)"
status: DONE
severity: High
category: bug
tags: [bpir, async-action, dedicated-k2-subclass, HasDedicatedAsyncNode, listen-gameplay-message]
---

# BPIR named-factory path can't wire async actions with `HasDedicatedAsyncNode`

The DONE fix [`B-bpir-asyncaction-subclass-missed`](B-bpir-asyncaction-subclass-missed.md) added a named-factory resolution path that creates a generic `UK2Node_AsyncAction` and calls `InitializeProxyFromFunction(Func)` before `AllocateDefaultPins`. That covers factories whose proxy class carries a **static** delegate signature (e.g. `UAsyncAction_ShowConfirmation::ShowConfirmationYesNo`). The same fix's IN-REVIEW note explicitly states the path opts out when the owner class carries `HasDedicatedAsyncNode` meta. This is the lane that's still broken.

`UAsyncAction_ListenForGameplayMessage::ListenForGameplayMessages` is the canonical case. Its delegate pin (`OnMessageReceived`) carries a `Payload` output whose type is determined dynamically from the factory's `PayloadType` UScriptStruct* argument. That dynamic layout lives on a dedicated K2 subclass `UK2Node_AsyncAction_ListenForGameplayMessages::AllocateDefaultPins` — the generic `UK2Node_AsyncAction` doesn't know how to produce it.

## Repro

Inside any Blueprint:

```bpir
%n3 = call ListenForGameplayMessages(WorldContextObject: self, Channel: (TagName="App.Notifications.GhostOpponentsSpawned"), PayloadType: /Script/PDSGame.LyraVerbMessage, MatchType: EGameplayMessageMatch::ExactMatch) [OnMessageReceived -> @h]

@h:
    call DoSomething()
```

Result: `Could not find exec output pin 'OnMessageReceived' on node`. Inspecting the created node via `blueprint.get_node_connections` shows `execOutputs: []`, `dataOutputs: []` — the generic `UK2Node_AsyncAction` was placed without any delegate-related pins.

Alternative attempt — manually create the dedicated subclass:

```
blueprint.graph.create_node {nodeType: "K2Node_AsyncAction_ListenForGameplayMessages", graphName: "EventGraph", x:..., y:...}
```

Result: node placed with title `"Async Task: Missing Function"`, all pin arrays empty. The dedicated K2 subclass needs `InitializeProxyFromFunction` (or its own equivalent) with the factory UFunction to materialize its pins, but `graph.create_node` only spawns the bare K2 class without that initialization step.

Decompile of an existing working example (`/App/App/UI/W_OutOfBoundsMessage`) shows the canonical text form:

```bpir
%n1: object<AsyncAction_ListenForGameplayMessage> = call K2Node_AsyncAction_ListenForGameplayMessages(Channel: (TagName="..."), PayloadType: ..., MatchType: ...) [OnMessageReceived -> @h]
```

Both `call ListenForGameplayMessages(...)` (named-factory) and `call K2Node_AsyncAction_ListenForGameplayMessages(...)` (explicit K2 class) need to produce a fully-initialized dedicated-subclass node. Today the first form falls back to generic and the second errors with `No async action factory named 'ListenForGameplayMessages' was found`.

## Impact

- The race analyzer's lap row can't auto-refresh on `App.Notifications.GhostOpponentsSpawned`. Currently only `OnInitialized` runs `RebuildLapRow`, so ghosts that spawn after the analyzer HUD opens are missed until the user closes and reopens it. Real-world impact is small because ghosts typically spawn at track BeginPlay, before the analyzer opens — but mid-session ghost arrival (live multiplayer joins, dynamic spawn) won't be reflected.
- Any BP that needs to react to a gameplay message via the canonical UE pattern is currently un-authorable through BPIR. Forces C++ side-channels or hand-edited graphs that don't round-trip.

## Workaround

None viable through BPIR. The dedicated subclass can be created with `graph.create_node` but its `Initialize` step is not exposed — the node stays in "Missing Function" state. A previous subagent suggested adding a `blueprint.graph.configure_async_node` RPC; that doesn't exist today.

Practical workaround for callers: skip the listener, rebuild only at construction time, accept the staleness.

## Fix

Both surfaces need to handle the `HasDedicatedAsyncNode` lane:

1. **Named-factory path** (`call FactoryName(...)`) — detect when the factory's owner class has `HasDedicatedAsyncNode` meta. Resolve the matching dedicated subclass `UK2Node_AsyncAction_<FactoryFunctionName>` via `FindObject<UClass>` / class iteration. Spawn that subclass instead of the generic `UK2Node_AsyncAction`. Then initialize it through whatever method that subclass uses (typically `SetCallbackFunction` / `InitializeAsyncAction` or `AllocateDefaultPins` after seeding the function pointer — depends on the subclass).

2. **Explicit-K2-class path** (`call K2Node_AsyncAction_<Name>(...)`) — currently fails to find a factory. The compiler should accept this form by treating the suffix after `K2Node_AsyncAction_` as the factory function name and routing identically to (1). The factory lookup should still happen because the dedicated subclass needs the same `InitializeProxyFromFunction(Func)` call to know which UFunction it represents.

3. **Decompiler round-trip** — `blueprint.decompile` already emits the explicit `K2Node_AsyncAction_<Name>(...)` form for dedicated subclasses. Once (1) and (2) compile, round-trip will work without further decompiler changes.

Regression coverage: add tests that compile both forms for `ListenForGameplayMessages` and at least one other `HasDedicatedAsyncNode` factory (e.g. anything in `OnlineSubsystem*`), then verify the resulting node has the expected delegate exec output pins (`OnMessageReceived`, payload data pin, etc.) and survives a `blueprint.compile`.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while wiring `App.Notifications.GhostOpponentsSpawned` listener in `W_AnalyzerControls::OnInitialized` to refresh the dynamic lap row on mid-session ghost spawn. Both `call ListenForGameplayMessages(...)` (generic node, zero exec outputs) and `graph.create_node K2Node_AsyncAction_ListenForGameplayMessages` (subclass spawned but "Missing Function") fail. Prior DONE fix `B-bpir-asyncaction-subclass-missed` explicitly opts out of `HasDedicatedAsyncNode` factories — that opt-out is the gap.
- `#2-dedicated-subclass-implemented` `IN-REVIEW` developer — Added a parallel dedicated-lane probe rather than relaxing `IsAsyncActionFactory`. New sibling helper `FCodeNodeEmitter::ResolveDedicatedAsyncActionSubclass(UFunction*)` returns the matching `UK2Node_AsyncAction_<FactoryFunctionName>` subclass when the factory's owner class carries the `HasDedicatedAsyncNode` UCLASS meta, using a hybrid `FindFirstObjectSafe<UClass>(name-convention)` + `TObjectIterator<UClass>` fallback. `CreateAsyncActionNode` gained an optional `UClass* NodeClass = nullptr` parameter so the same exec-wiring path serves both lanes via `NewObject<UK2Node_AsyncAction>(Graph, SpawnClass)`. The generic-fallback short-circuit in `CreateGenericK2Node` for `UK2Node_AsyncAction`-derived classes now carries a bypass-contract comment. Three call sites in `BpirCompiler.cpp` probe the dedicated lane first: the named-factory dispatch (~line 4991), the `EmitGenericK2NodeInstruction` explicit-K2-class branch (~line 7180) which adds a `UBlueprintAsyncActionBase`-filtered iterator to recover the factory rejected by `IsAsyncActionFactory`'s opt-out, and the shared `EmitAsyncActionFactoryNode` lambda (now takes `UFunction*, UClass*`). Regression test `FAsyncActionDedicatedSubclassRoundTripTest` covers `ListenForGameplayMessages`, asserting (a) runtime UClass is the dedicated subclass, (b) factory identity, (c) `Payload` pin materializes, (d) compile→decompile→recompile preserves the dedicated UClass; the existing `FAsyncActionFactoryIdentityRoundTripTest` was strengthened with a runtime-UClass assertion for the generic lane so regressions in either direction are catchable. Regression test covers only `ListenForGameplayMessages` — `GameplayMessageRouter` exposes only that one `HasDedicatedAsyncNode` factory, and engine-shipped factories with the meta require optional modules not linked by this plugin. Fallback authorized by the plan's `Unresolved Questions` clause (single-factory regression with documented fallback).
- `#3-verified-listener-pins-materialize` `DONE` tester — Recompiled `W_AnalyzerControls::OnInitialized` with `call ListenForGameplayMessages(WorldContextObject: self, Channel: (TagName="App.Notifications.GhostOpponentsSpawned"), PayloadType: /Script/PDSGame.LyraVerbMessage, MatchType: EGameplayMessageMatch::ExactMatch) [OnMessageReceived -> @h]` → `compiled: true, status: UpToDate`. Verified via `blueprint.get_node_connections` that the resulting node titled `ListenForGameplayMessages` carries the `OnMessageReceived` exec output pin wired to `RebuildLapRow`, plus `WorldContextObject` data input wired to self. No "Could not find exec output pin" error from the prior session, no "Missing Function" title. Race analyzer lap row now refreshes on mid-session ghost spawn.
