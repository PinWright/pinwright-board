---
id: B-bpir-widget-event-at-syntax-silent-degrade
title: "BPIR `entry event @Widget:Event` parses but silently produces an unbound K2Node_Event"
status: DONE
severity: High
category: bug
tags: [bpir, widget-event, component-bound-event, silent-degrade, parser-permissiveness]
---

# BPIR `entry event @Widget:Event` parses but silently produces an unbound K2Node_Event

The BPIR parser accepts `entry event @WidgetName:EventName(...)` (colon-prefixed `@`-style widget binding) and runs it through compilation, producing a `K2Node_Event` titled `Event @WidgetName:EventName` in the EventGraph. The node is **never bound** to the widget's actual delegate (e.g. `RenameButton.OnClicked`). It just sits there as a plain custom-shaped event, dead at runtime.

Compile reports `success: true`, `compiled: true`, `status: "UpToDate"`, and creates the full downstream chain (10+ nodes per entry). There is no warning about the unrecognized syntax — the documented form is `entry widget_event WidgetName.OnClicked()` (period, not colon, and `widget_event` keyword instead of plain `event`), but the colon-`@` form is silently accepted.

## Repro (observed this session, `W_MyReplayListItem`)

Trying to mirror the existing `DeleteButton` bound-event pattern for `RenameButton` and `RenameButtonHovered`:

```
entry event @RenameButton:CommonButtonBaseClicked(object<CommonButtonBase> Button) {
    %n0 = call GetPlayerController(PlayerIndex: 0)
    %n1 = call GetLocalPlayerFromController(PlayerController: %n0)
    %n2 = call PushContentToLayer_ForPlayer(LocalPlayer: %n1, LayerName: (TagName="UI.Layer.Menu"),
            WidgetClass: /App/App/UI/LobbyAndMenu/Popups/W_RenameReplay.W_RenameReplay_C)
    %n3 = cast<W_RenameReplay_C>(%n2) [success -> @ok]
@ok:
    set %n3.AsWRenameReplay.Replay = $Replay
    set %n3.AsWRenameReplay.ParentScreen = $ParentScreen
    call ApplyReplay(Target: %n3.AsWRenameReplay, Replay: $Replay)
}
```

Compile result: `success: true`, 10 nodes created, no warnings.

`mcp__editor_automation__.call path="blueprint.graph.get_execution_flow" args={"includeAllEntryPoints":true,"entryPointsOnly":true}` showed the new entry as `K2Node_Event` titled `Event @RenameButton:CommonButtonBaseClicked` — NOT `K2Node_ComponentBoundEvent` like the working `On Clicked (DeleteButton)` sibling. Pin list confirms: only `OutputDelegate` + `then` outputs, missing the `Button` output that `K2Node_ComponentBoundEvent` exposes.

Repro is 100% reliable across both event variants (`RenameButton` + `RenameButtonHovered`).

## Impact

The wrong-syntax events compile cleanly, look right in the entry-point list at first glance, and never fire at runtime. With BP-only inspection a user might not notice for a long time — only when clicking the button does nothing in PIE.

In this session it cost ~14 nodes of cleanup work (2 entry points + 12 chain nodes deleted via `blueprint_graph_delete_node` + `blueprint_graph_find_orphaned_nodes` sweep) once the symptom was understood. Verifying the fix (re-emit with `entry widget_event RenameButton.OnClicked()`) produced the correct `K2Node_ComponentBoundEvent` titled `On Clicked (RenameButton)`.

**Workaround:** use `entry widget_event WidgetName.OnClicked()` — the documented form per `bpir-language-reference.md:82`.

**Proposal:** parser should reject the `@Widget:Event` shape with a hard parse error pointing at `widget_event`. Alternatively, treat `@Widget:Event` as an alias and route it through the same `widget_event` codepath. Either way, silent degradation to an unbound `K2Node_Event` is the worst outcome — the user is told it worked and gets dead nodes.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while wiring rename buttons on `W_MyReplayListItem`. Two `entry event @RenameButton:Common...` and `entry event @RenameButtonHovered:Common...` blocks compiled clean (status UpToDate, 20 nodes), but produced `K2Node_Event` (plain) instead of `K2Node_ComponentBoundEvent`. Cleaned up by deleting all 14 nodes (2 entries + 12 chain) and re-emitting with `entry widget_event RenameButton.OnClicked()` / `entry widget_event RenameButtonHovered.OnClicked()` — the correct form produced bound events titled "On Clicked (RenameButton)" / "On Clicked (RenameButtonHovered)" with the `Button` output pin present.
- `#2-reject-at-colon-event-name` `IN-REVIEW` developer — Added identifier-shape validation in `BpirParser.cpp::ParseEntryLine` so `entry event @Widget:Event(...)` now fails parsing with a helpful pointer to `widget_event`. Regression test `FBpirParserRejectAtColonEventNameTest` in `TestBpirParser.cpp` asserts the parser rejects the bad form while still accepting the documented `entry widget_event Widget.OnClicked()` form.
- `#3-verified-rejection` `DONE` tester — Verified on `/Game/App/UI/Test/W_McpVerifyTemp`: `compile_bpir` body `entry event @Foo:Bar() {}` returns `COMPILE_FAILED: Line 1: Invalid event name '@Foo:Bar'. Did you mean 'entry widget_event Foo.Bar()'? The '@Name:Event' form is not supported — use 'widget_event' for widget delegate bindings.` Hard parse error with a precise migration hint exactly as designed; no silent-degrade path remains.
