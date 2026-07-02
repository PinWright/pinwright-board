---
id: F-activatable-push-by-layer-tag
title: "`ui.activatable_push` should accept a CommonUI layer gameplay tag (+ player index) and resolve the PrimaryGameLayout host/stack itself"
status: OPEN
severity: Medium
category: feature
tags: [ui, activatable_push, common-ui, common-game, primary-game-layout, layer-tag, gameplay-tags]
---

# `ui.activatable_push` should accept a layer gameplay tag and resolve the host/stack itself

`ui.activatable_push` (and the sibling `activatable_pop` / `list_stack_widgets` /
`get_active_widget`) address the target as `host` (runtime instance name of the layout
widget, e.g. `W_OverallUILayout_C_0`) + `stack` (container child name, e.g. `Menu_Stack`)
— see `FindStackInPie` at `Source/PinWright/Private/Handlers/UI/UiActivatableStackHandler.cpp:17-58`
(iterates live `UUserWidget`s by `GetName()==host`, then `GetWidgetFromName(stack)` cast to
`UCommonActivatableWidgetContainerBase`). Both are implementation details of the layout asset.
Games built on CommonGame/Lyra address layers by GAMEPLAY TAG (`UI.Layer.Menu`, `.Modal`,
`.GameMenu`, `.Game`) via `UCommonUIExtensions::PushContentToLayerForPlayer(LocalPlayer, Tag, Class)`
/ `UPrimaryGameLayout::GetLayerWidget(Tag)`.

Discovering host/stack names required custom python this session: iterate `PrimaryGameLayout`
widgets for the instance name, then `ObjectIterator(CommonActivatableWidgetContainerBase)` filtered
by outer to learn the child stack names (`Modal_Stack`, `Menu_Stack`, `GameMenu_Stack`,
`GameLayer_Stack`). Direct python replication of the tag path is also blocked for agents:
`FGameplayTag('UI.Layer.Menu')` can't be built in UE 5.7 python (`request_gameplay_tag` not
exposed; `MakeLiteralGameplayTag` takes a tag; `TagName` read-only). Proposal: accept `layerTag`
(+ optional `playerIndex`, default 0) as an alternative to `host`/`stack` on all four
`ui.activatable_*` methods, resolving to the same `UCommonActivatableWidgetContainerBase` the
current path returns.

**Dependency note (verified):** `UPrimaryGameLayout` / `UCommonUIExtensions` live in the project's
`Plugins/CommonGame` (`COMMONGAME_API`, Lyra-only), which PinWright deliberately does **not** link —
`Build.cs` links `CommonUI` + `GameplayTags` + `UMG` only, and CLAUDE.md states CommonGame/UIExtension
are intentionally not deps (linking them would break non-Lyra hosts). So the branch cannot call
`GetLayerWidget`/`GetPrimaryGameLayout` directly — both are plain C++ methods, **not** UFUNCTIONs, so
`ProcessEvent` reflection won't reach them either. But `UPrimaryGameLayout::Layers` is a
`UPROPERTY(Transient) TMap<FGameplayTag, TObjectPtr<UCommonActivatableWidgetContainerBase>>` and IS
reflectable. FGameplayTag construction is trivial natively (GameplayTags is linked;
`FGameplayTag::RequestGameplayTag` — the python blocker does not apply to a C++ handler).

**Workaround:** Discover `host` via an ObjectIterator over live `PrimaryGameLayout` widgets, discover
the per-layer `stack` child names via `ObjectIterator(UCommonActivatableWidgetContainerBase)` filtered
by outer, then call `ui.activatable_push {host, stack, widgetClass}` as today.

**Fix:** Add optional `layerTag` (+ `playerIndex`, default 0) as an alternative to `host`/`stack`.
Resolve by reflection so no CommonGame link is added: `FindObject<UClass>(nullptr,
"/Script/CommonGame.PrimaryGameLayout")`, find the live instance owned by the target `playerIndex`
local player (extend `FindStackInPie` / `TObjectIterator<UUserWidget>` filtered by `IsA(that UClass)`),
then read its `Layers` `FMapProperty` and look up the `UCommonActivatableWidgetContainerBase` by the
requested `FGameplayTag` (built via `FGameplayTag::RequestGameplayTag`). Feed the resolved container
into the existing `AddWidgetInstance` path. Error gracefully when CommonGame is absent
(class not found) or the tag isn't registered (e.g. `LAYER_HOST_UNAVAILABLE` / `LAYER_NOT_FOUND`);
keep `host`/`stack` working unchanged, `layerTag` mutually exclusive with them.

## History
- `#1-filed-layer-tag-addressing` `OPEN` reporter — Filed as an addressing-model refinement of the DONE `F-ui-common-activatable-stack` (which shipped the host/stack surface), distinct from the IN-REVIEW docs tickets `E-activatable-host-runtime-instance-name-undocumented` (host-naming friction) and `E-activatable-push-requires-c-suffix` (widgetClass `_C` resolution) — this proposes a NEW input (`layerTag`+`playerIndex`), not a doc/resolver fix on the existing params. Session evidence: three failed python attempts to build `FGameplayTag('UI.Layer.Menu')` (`request_gameplay_tag` AttributeError; `MakeLiteralGameplayTag` TypeError; `TagName` read-only), then an ObjectIterator discovery dance for host + stack child names, then `ui.activatable_push {host:"W_OverallUILayout_C_0", stack:"Menu_Stack", widgetClass:"/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone.W_DroneSelect_EditDrone"}` → success `{instanceName:"W_DroneSelect_EditDrone_C_0"}`. Verified dependency picture: `UPrimaryGameLayout` is `COMMONGAME_API` in `Plugins/CommonGame/Source/Public/PrimaryGameLayout.h`; PinWright's `Build.cs` links `CommonUI`/`GameplayTags`/`UMG` but not CommonGame/UIExtension; `GetLayerWidget`+static accessors are non-UFUNCTION (not ProcessEvent-reflectable) but the `Layers` UPROPERTY map is — so a reflection-based layerTag branch is feasible without a new hard link.
