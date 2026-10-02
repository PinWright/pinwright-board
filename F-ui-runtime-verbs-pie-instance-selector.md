---
id: F-ui-runtime-verbs-pie-instance-selector
title: "`ui.create_hud` / `ui.activatable_*` cannot target a chosen PIE instance in multi-client PIE (no `world: \"client:N\"` selector like `editor.console_command`)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [ui, create_hud, activatable_push, pie, multiplayer, listen-server, world-selector]
encounters: 1
lastSeen: 2026-09-30T09:00:00Z
---

# Runtime UI verbs cannot target a specific PIE instance

In a multi-instance PIE (`editor.play {numClients: 2, netMode: "listen"}`), the task was to open the same
activatable results screen on the listen server **and** on the client, then read what each showed. No `ui.*`
verb could do it:

- `ui.create_hud` creates the widget in `GEngine->GameViewport->GetWorld()`
  (`Source/PinWright/Private/Handlers/UI/UiHandler.cpp`, `ui.create_hud`). That is whichever PIE viewport the
  engine currently holds, and it passes no owning player.
- `ui.activatable_push` / `ui.list_stack_widgets` with `layerTag` resolve the layout through
  `ResolveStackByLayerTagInPie` (`Source/PinWrightCommonUI/Private/Handlers/UI/ActivatableLayerResolver.cpp`).
  That code matches `GetOwningLocalPlayer()->GetLocalPlayerIndex() == playerIndex`. Every PIE game instance has
  its own local player 0, so `playerIndex: 0` matches whichever `UPrimaryGameLayout` `TObjectIterator` yields
  first, silently.
- The `ui.set_widget_*` setters resolve through `ResolveQueryWorld("auto")`: the PIE world first, again with no
  instance choice.

`editor.console_command` and `editor.pie_status` already have the vocabulary (`server`, `client:N`, `pie:N`).
The `ui.*` runtime verbs don't accept it.

The Python fallback is closed too. UE 5.8 exposes `UWidgetBlueprintLibrary` to Python as `unreal.WidgetLibrary`
(its ScriptName; `unreal.WidgetBlueprintLibrary` does not exist). `Create` is `BlueprintInternalUseOnly`, so
Python cannot construct a UserWidget for a chosen player controller. This is engine behavior, not a PinWright
defect, but it means the `ui.*` verbs are the only in-API route.

**Workaround used:** find a game-side widget that already lives in each instance (via `unreal.ObjectIterator`
over its class, filtered by `get_owning_player()`), and `call_method` its own "show screen" function per
instance. This works only when the game has such a function.

**Proposed:** accept the same optional `world` selector (`server` / `client:N` / `pie:N`) on `ui.create_hud`,
`ui.activatable_push`, `ui.activatable_pop`, `ui.list_stack_widgets`, `ui.get_active_widget` and the
`ui.set_widget_*` setters. Resolve it to that instance's world and its first local player's controller, pass
that controller as the owning player in `ui.create_hud`, and echo `pieInstance` in the response. With more than
one PIE instance and no selector, `ResolveStackByLayerTagInPie` should return an ambiguity error instead of
taking the first layout it finds.

## History
- `#1-listen-pie-results-screen` `OPEN` reporter - Filed from a PDS race-results repro, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin source `2580e7f4`. The task was a 2-instance listen PIE, opening `W_RaceOnlineResultsFrame` on the host and on the client and reading each one's rows. Reading `ui.create_hud` and `ui.activatable_push` showed no per-instance targeting, so I did not use them. Python `unreal.WidgetBlueprintLibrary.create` / `unreal.WidgetLibrary.create` both failed (`AttributeError`). Worked around by calling the game's own `W_RaceResultsHandler.ShowOnlineRaceFrame` in each instance through `unreal.ObjectIterator` + `call_method`. Costly: about 6 extra calls and a source dive into `W_RaceResultsHandler`.
- `#2-world-selector-on-ui-runtime-verbs` `IN-REVIEW` developer - Added the optional `world` selector (`server` / `client` / `client:N` / `pie:N`, the `editor.console_command` grammar and the `world` name drive.* uses) to `ui.create_hud`, `ui.set_widget_text` / `_image` / `_visibility`, `ui.remove_widget_from_viewport`, `ui.activatable_push` / `_pop`, `ui.list_stack_widgets`, `ui.get_active_widget`. One shared resolver: new pure `PieWorldSelector::ResolveGameWorld` in `Source/PinWright/Private/Handlers/Editor/PieWorldSelector.h` (explicit selector -> that PIE world or `WORLD_NOT_FOUND`; omitted -> `ResolveOmitted`: no PIE keeps the old default, one PIE world is used, several -> `TARGET_AMBIGUOUS`; `editor` / malformed -> `INVALID_ARGUMENT`). `ui.create_hud` creates in the selected world (CreateWidget(World) owns it by that world's first local player) instead of `GEngine->GameViewport`, `NO_VIEWPORT` on a dedicated server; setters and named removal search only that world; `ResolveStackByLayerTagInPie` and the host+stack lookup filter by the selected world (`Source/PinWrightCommonUI/Private/Handlers/UI/ActivatableLayerResolver.{h,cpp}`, `UiActivatableStackHandler.cpp`), so `playerIndex: 0` no longer matches whichever instance's layout comes first. Successes echo `pieInstance` + `kind`. Docs: `docs/wiki-src/ui.md` (new `## Multi-client PIE` section), CHANGELOG. Tests: `PinWright.ui.pie_world.ResolveGameWorldContract` (resolver over synthetic listen-server + client contexts), `PinWright.ui.pie_world.RuntimeVerbsRouteSelector` and `PinWright.ui.activatable.WorldSelectorRouted` (each verb, both activatable addressing modes, routes `world` into the resolver: `WORLD_NOT_FOUND` / `INVALID_ARGUMENT` with no PIE; before the fix `world` was ignored and they answered `NO_VIEWPORT` / `WIDGET_NOT_FOUND` / `LAYER_HOST_UNAVAILABLE` / `HOST_NOT_FOUND`). Multi-client PIE is not started in automation, so actual per-instance routing in a live listen-server session is unverified; a tester should run the #1 repro (2-instance listen PIE, `ui.create_hud {world:"server"}` and `{world:"client:1"}`, then `ui.activatable_push {layerTag, world:"client:1"}`).
- `#3-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.ui.pie_world.ResolveGameWorldContract` (resolver over synthetic listen-server and client contexts), `PinWright.ui.pie_world.RuntimeVerbsRouteSelector` and `PinWright.ui.activatable.WorldSelectorRouted` (every verb routes `world` into the resolver; with no PIE: `WORLD_NOT_FOUND` / `INVALID_ARGUMENT`). Not demonstrated: per-instance routing in a live listen-server session, which is the whole ask; automation starts no multi-client PIE. Needs: the #1 repro (2-instance listen PIE, `ui.create_hud {world:"server"}` and `{world:"client:1"}`, then `ui.activatable_push {layerTag, world:"client:1"}`), each landing in its own instance with `pieInstance` echoed.
