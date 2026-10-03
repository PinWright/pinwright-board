---
id: B-graph-create-node-target-ignored-for-variableget
title: "blueprint.graph.create_node ignores the documented `target` slot for VariableGet/VariableSet and fails with VARIABLE_NOT_FOUND: Variable '' — only the legacy `variableName` alias works"
status: DONE
severity: Medium
category: bug
tags: [blueprint, blueprint-graph, create_node, variableget, variableset, target, alias, docs-mismatch]
encounters: 4
lastSeen: 2026-09-29T16:25:00Z
---

# `target` is documented for `VariableGet` and silently unused

## Repro

```
blueprint.graph.create_node {assetPath:"/Game/FPS/UI/Test/BP_HUDTestPawn",
  graphName:"GetHealth", nodeType:"VariableGet", target:"Health01", x:16, y:128}
-> [VARIABLE_NOT_FOUND] Variable '' not found
```

Five calls of that shape (variables `Health01`, `Spread`, `WeaponNameVar`, `MagAmmo`,
`ReserveAmmo`, all existing and all listed by `blueprint.inspect`) failed identically. The empty
quotes in `Variable '' not found` are the tell: the handler read its variable-name slot, found
nothing there, and never looked at `target`.

Swapping one field, everything else identical, works:

```
blueprint.graph.create_node {assetPath:"...", graphName:"GetHealth",
  nodeType:"VariableGet", variableName:"Health01", x:16, y:128}
-> {"nodeId":"DC5994A94A275BED4A2E91BD4ACDBE5E","nodeName":"K2Node_VariableGet_0"}
```

## The wiki says `target` should work

`Saved/PinWright/wiki/blueprint.graph.create_node.md` lists `VariableGet` / `K2Node_VariableGet`
and `VariableSet` / `K2Node_VariableSet` first under **"Supported target-aware node types"**, gives
the rule under **"`target` interpretation by type"** — *"CallFunction / VariableGet / VariableSet /
Event — function, variable, or event name"* — and even carries a worked example:

```json
{ "assetPath": "/Game/BP_Foo.BP_Foo", "nodeType": "VariableGet", "target": "CurrentLap", "x": 300, "y": 200 }
```

That example does not work. The page then describes `variableName` as a **legacy alias** accepted
"when `target` is absent", which inverts the actual behaviour: `variableName` is the only field
that works and `target` is the one that is ignored.

## What should happen

`target` should resolve the variable for `VariableGet` / `VariableSet`, matching its own
documentation and the behaviour of the other target-aware types. Failing that, the page must be
corrected and the handler must reject an unusable `target` explicitly rather than reporting a
missing name it was never given — `VARIABLE_NOT_FOUND: Variable ''` sends the caller looking for a
missing variable instead of a misread parameter.

Worth checking whether `CallFunction` and `Event` have the same gap; I only exercised `VariableGet`.

**Workaround:** use `variableName` for variable nodes.

severity rationale: impact=soft blocker — the documented parameter is inert and the error
misdirects, but the legacy alias works and is on the same page x reach=`create_node` is the
surgical-edit path used whenever `compile_bpir` cannot reach a graph (e.g. Blueprint Interface
implementation graphs, which is exactly where I needed it) -> Medium.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) while implementing `BPI_HUDSource` on `/Game/FPS/UI/Test/BP_HUDTestPawn`. `blueprint.compile_bpir` cannot author a Blueprint Interface implementation graph at all (`Found more than one function with the same name` when the interface graphs exist; `INTERFACE_MUTATION_FAILED` if you create the functions first — see `B-interface-function-with-outputs-unimplementable`), so `blueprint.graph.create_node` + `connect_pins` into the existing interface graph was the only route left. Five `target:"<VarName>"` calls returned `VARIABLE_NOT_FOUND: Variable ''`; the same five with `variableName:"<VarName>"` all returned node ids, and the resulting graphs compile and decompile correctly. The wiki page lists `VariableGet` under "Supported target-aware node types" with a worked `target` example, and calls `variableName` a legacy alias — the opposite of what the handler does.
- `#2` `OPEN` reporter — Hit again on EAContentExamples58 (UE 5.8), same shape, three days later: `create_node {graphName:"StartFire", nodeType:"VariableGet", target:"bTriggerHeld", x:1760, y:216}` -> `[VARIABLE_NOT_FOUND] Variable '' not found`; the identical call with `variableName:"bTriggerHeld"` returned `{"nodeId":"7C0489D94CA2B06C0A78AD92130F2FF5"}`. **Answers the ticket's open question: `CallFunction` does NOT have the gap.** In the same session `create_node {nodeType:"CallFunction", target:"KismetMathLibrary::Subtract_IntInt"}` and `{nodeType:"CallFunction", target:"Actor::GetInstigatorController"}` both resolved and returned node ids, and `VariableSet` needed `variableName` on all four of its uses (`ShotCounter`, `PenetrationsLeft`, `CurrentShotOrigin`). So the defect is scoped to the variable node branch, not to `target` parsing in general — which makes the fix a one-branch change and the wiki page wrong only in its variable rows.
- `#3-external-owner-qualified-target` `OPEN` reporter - Seen again, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`, on `/App/App/UI/LobbyAndMenu/Elements/W_OnlineUsersListItem` `EventGraph`. `create_node {nodeType:"VariableGet", target:"OpponentInfo::PlayerState"}` and `target:"OpponentInfo.PlayerState"` both returned `VARIABLE_NOT_FOUND: Variable '' not found` (the qualified form is the documented way to name an external owner). The legacy `{variableName:"PlayerState", memberClass:"/Script/App.OpponentInfo"}` returned `Variable 'PlayerState' not found`, so an external-class member getter cannot be created with this verb at all. Workaround: `replace_node` an existing getter on the same owner with `target:"OpponentInfo::PlayerState"` (works, but see B-replace-node-drops-external-self-wire). Cheap (three calls).
- `#4-event-target-also-ignored` `OPEN` reporter - Answers the open question in the body: `Event` has the same gap. UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`: `create_node {assetPath:"/App/App/UI/LobbyAndMenu/TrackMenu/W_GameEventsHandler", nodeType:"Event", target:"Destruct", x:0, y:2900}` returned `INVALID_ARGUMENT: eventName required`; the same call with `eventName:"Destruct"` worked. `CallFunction` with `target:"AppUIExtensions::CancelGameplayMessageListeners"` did work. Cheap (one retry).
- `#5-target-honoured` `IN-REVIEW` developer - Still reproducible at plugin `10212ee4`: the VariableGet/VariableSet branches of `blueprint.graph.create_node` read only `variableName`, Event only `eventName`; CustomEvent and Cast (also listed as target-aware on the wiki) read only `eventName` / `targetClass`. Fix in `Handlers/Blueprint/BlueprintGraphCrudHandler.cpp`: the two variable branches are merged and resolve through the shared `ParseGraphTargetSpec` + `ResolveGraphVariableTarget` + `DoesGraphVariableExist` helpers `replace_node` already uses - `target` wins over `variableName`, `memberClass` supplies the owner when target is unqualified, an owner that is not this BP or an ancestor creates an external-member node (answers #3: `OpponentInfo::PlayerState` and `{variableName, memberClass}` now both work), the not-found message names the owner, and a missing name is `INVALID_ARGUMENT: target required ...` instead of `VARIABLE_NOT_FOUND: Variable ''`. Event/CustomEvent/Cast read `target` first (Event's qualifier fills memberClass; CustomEvent drops it). Param descriptions, `docs/wiki-src/blueprint.graph.md` alias paragraph, CHANGELOG updated. Test `PinWright.blueprint.graph.create_node.TargetParamResolvesVariableEventAndCastNodes` (`Tests/Blueprint/TestBlueprintGraphCreateNodeTargetAware.cpp`) creates VariableGet/VariableSet (self), VariableGet `ActorComponent::ComponentTags` (external), Event `ReceiveDestroyed`, CustomEvent and Cast `Pawn` via `target` only, then asserts the no-name INVALID_ARGUMENT and the named VARIABLE_NOT_FOUND failure directions.
- `#6-review-fixes` `IN-REVIEW` developer - Review SHOULD-FIX fixed: with `memberClass` (or a qualified `target`) naming the Blueprint's own class, the existence check looked only at `OwnerClass->FindPropertyByName`, so a variable added but not yet compiled (in `NewVariables` / the skeleton only) got `VARIABLE_NOT_FOUND`, a regression for the legacy `{variableName, memberClass}` form. `bSelfMember` is now computed first and a self member is checked with no owner (`NewVariables` + generated + skeleton class). The variable target resolution and all its refusals (`INVALID_ARGUMENT`, `CLASS_NOT_FOUND`, `VARIABLE_NOT_FOUND`) moved before the `FScopedTransaction` / `Blueprint->Modify()`, so a refused call no longer dirties the Blueprint. Test `PinWright.blueprint.graph.create_node.TargetParamResolvesVariableEventAndCastNodes` gains the legacy `{variableName, memberClass:<GeneratedClass path>}` case on a freshly added, uncompiled variable (precondition-asserted absent from `GeneratedClass`); helper comment corrected (it calls the handler, not the dispatcher). CHANGELOG updated. Re-review NIT: an ancestor qualifier (`Actor::BpOnlyVar`) must also name a property of that ancestor, else VARIABLE_NOT_FOUND (asserted in the same test).
- `#7-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.blueprint.graph.create_node.TargetParamResolvesVariableEventAndCastNodes`. Acceptance met: (1) `target` alone creates VariableGet and VariableSet self-member nodes (the #1/#2 repro); (2) a qualified external owner `ActorComponent::ComponentTags` creates an external-member getter (the #3 `OpponentInfo::PlayerState` shape on a native class); (3) Event, CustomEvent and Cast resolve through `target` (#4); (4) a missing name now returns `INVALID_ARGUMENT` instead of `VARIABLE_NOT_FOUND: Variable ''`, and a wrong name returns VARIABLE_NOT_FOUND naming the owner; (5) the legacy `{variableName, memberClass}` form still works on an uncompiled self variable (review fix #6). Coverage limit: not re-run against the reporter's `/App` W_OnlineUsersListItem or EAContentExamples58 assets.
