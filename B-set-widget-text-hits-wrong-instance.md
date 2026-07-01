---
id: B-set-widget-text-hits-wrong-instance
title: "`ui.set_widget_text` reports success but writes to the wrong UserWidget instance (editor-world first), leaving the painted PIE HUD unchanged"
status: IN-REVIEW
severity: High
category: bug
tags: [ui, set_widget_text, pie, silent-noop]
---

# `ui.set_widget_text` reports success but writes to the wrong UserWidget instance

`ui.set_widget_text` is documented as targeting "the **live** runtime instance
… live in the viewport". In practice it returns `{key,value}` success while the
text never appears on the painted PIE widget. The set lands on a *different*
`UserWidget` instance than the one rendered in the viewport, so the call is a
**silent success-with-no-effect** from the caller's point of view: the success
payload echoes back the value, but every live read-back (`widget.describe` /
`widget.export_xml` with `capture_source=live`) shows `Text: ""` on the painted
instance — even with PIE **paused** (so it is not a gameplay-Tick reset race).

## Root cause

`Handlers/UI/UiHandler.cpp` `ui.set_widget_text` builds its candidate widget
list **editor-world first, PIE-world second**, then iterates and **breaks on the
first matching TextBlock**:

```cpp
UWidgetBlueprintLibrary::GetAllWidgetsOfClass(
    GEditor->GetEditorWorldContext().World(), Widgets, UUserWidget::StaticClass(), false);   // editor world FIRST
if (GEngine && GEngine->GameViewport && GEngine->GameViewport->GetWorld())
    UWidgetBlueprintLibrary::GetAllWidgetsOfClass(
        GEngine->GameViewport->GetWorld(), Widgets, UUserWidget::StaticClass(), false);      // PIE world appended after

for (UUserWidget* Widget : Widgets)
{
    UWidget* Child = Widget->GetWidgetFromName(FName(*Key));
    if (UTextBlock* TextBlock = Cast<UTextBlock>(Child))
    {
        TextBlock->SetText(FText::FromString(Value));
        bFound = true; break;   // breaks on FIRST match -> an editor-world (non-painted) instance wins
    }
}
```

When a `UserWidget` of the same class also exists in the editor world (preview /
editor-spawned instance), its `InteractionText` matches first and receives the
`SetText`, so the painted PIE instance — the one `ui.create_hud` added to the
viewport and the one `ui.screenshot` renders — is never touched. `bFound` is
true, so the handler reports success. Same editor-world-blind-to-PIE pattern as
the (fixed) `B-inspect-misses-pie-world`, but in `UiHandler.cpp`, which that fix
did not touch. `ui.set_widget_visibility` / `ui.set_widget_image` use a
`TObjectIterator` + first-`GetWorld()` match instead, so they are susceptible to
the same wrong-instance ordering and should be audited together.

## Verbatim repro (replayed live via mcp__editor-automation__call)

1. `editor.play {}` → `{"success":true}`
2. `ui.remove_widget_from_viewport {key:""}` → `{"removedCount":1}` (clear the
   auto-spawned BeginPlay HUD so there is exactly ONE painted instance)
3. `ui.create_hud {widgetPath:"/Game/Global/Blueprints/WBP_PlayerHUD.WBP_PlayerHUD_C"}`
   → `{"widgetName":"WBP_PlayerHUD_C_1"}`
4. `ui.set_widget_text {key:"InteractionText", value:"SCORE: 12500"}`
   → `{"key":"InteractionText","value":"SCORE: 12500"}`  (reports success)
5. `widget.describe {capture_source:"live", widgetName:"InteractionText", verbose:true}`
   → painted instance `owning_user_widget:"WBP_PlayerHUD_C_1"` has
   `"Text": ""` (the SCORE value did NOT land)
6. `editor.pause {}` → paused; re-run step 4 (success again); re-run step 5 →
   still `"Text": ""` on `WBP_PlayerHUD_C_1`. Paused rules out a Tick reset:
   the write simply never reaches the painted instance.

## Impact

Any "instantiate a HUD in PIE and populate one of its TextBlocks" task (a core
advertised use of `ui.create_hud` + `ui.set_widget_text`) silently fails: the
screenshot/reference frame shows empty text while the tool reports success, so
the caller has no signal anything is wrong. Needing to read `UiHandler.cpp` to
explain a success-but-no-effect call is itself a discoverability gap.

**Workaround:** none via the ui.* surface. The only reliable channel observed is
asset-side text (`property.set` on the widget BP CDO) before PIE, which does not
serve the "live runtime HUD with sample state" use case.

**Fix:** Resolve the target world PIE-first (mirror `McpActorUtils::ResolveQueryWorld`
added for `B-inspect-misses-pie-world`): when a PIE world exists, restrict the
search to it (or prefer the painted/top-level viewport instance) so the write
lands on the rendered widget. Echo the resolved `owning_user_widget` /
world in the response so callers can confirm which instance was mutated, and
treat "no painted instance matched" as an error rather than silently succeeding
on an editor-world decoy. Audit `ui.set_widget_visibility` and
`ui.set_widget_image` for the same first-match ordering.

## History
- `#3-additional-multi-instance-pie` `IN-REVIEW` reporter — Additional evidence: the `#2-pie-first-resolve` fix is INCOMPLETE — it resolves editor-world-vs-PIE-world but NOT multiple UserWidget instances within the SAME PIE world, so the bug still reproduces live. Replayed via mcp__editor-automation__call: `editor.play {}` → success; the game auto-spawns its own `WBP_PlayerHUD` at BeginPlay (`WBP_PlayerHUD_C_0`); then `ui.create_hud {widgetPath:"/Game/Global/Blueprints/WBP_PlayerHUD"}` → `{"widgetName":"WBP_PlayerHUD_C_1"}` (the PAINTED instance the viewport/screenshot renders). `ui.set_widget_text {key:"InteractionText", value:"Press E to open the door"}` returned success but echoed `"owning_user_widget":"WBP_PlayerHUD_C_0"` — the AUTO-SPAWNED instance, NOT the painted `_C_1`. Live read-back `widget.describe {capture_source:"live", instance_name:"WBP_PlayerHUD_C_1", widgetName:"InteractionText"}` shows the painted instance's `InteractionText` TextBlock has NO `Text` field at all (still empty/default — the prompt never landed). Same for visibility: `ui.set_widget_visibility {key:"Crosshair", visibility:"Collapsed"}` → success echoing `"owning_user_widget":"WBP_PlayerHUD_C_0"`, while live describe of painted `_C_1` Crosshair still reads `Visibility:"HitTestInvisible"` (its painted state — the collapse never landed on the rendered instance). Root cause survives: with both `_C_0` and `_C_1` in the resolved PIE world, `CollectRuntimeUserWidgets` → `GetAllWidgetsOfClass` returns the auto-spawned `_C_0` first and `FindRuntimeChildByKey`/`FindRuntimeWidgetByKey` take the FIRST match, so the write hits `_C_0` while `ui.create_hud`'s painted `_C_1` is left untouched. The `#2` regression test (`HitsResolvedWorldInstance`) only exercises a SINGLE widget in the no-PIE editor world, so it passes while this multi-instance-PIE path is broken. There is NO instance-targeting param on the `ui.set_widget_*` surface (no `instance_name`/root selector like `widget.describe capture_source=live` accepts) to redirect the write to the painted `ui.create_hud` instance, so there is no in-API workaround. Fix should: prefer the viewport-painted/top-most UserWidget (e.g. the instance `ui.create_hud` added to the viewport, or the GameViewport's added widgets) over an arbitrary first `GetAllWidgetsOfClass` hit, AND/OR accept an optional `instance_name` on `ui.set_widget_text` / `ui.set_widget_visibility` / `ui.set_widget_image` to disambiguate; and add a multi-instance-PIE regression test (two same-class UserWidgets, assert the painted one is written).
- `#2-pie-first-resolve` `IN-REVIEW` developer — Fixed the editor-world-first wrong-instance write in `Source/EditorAutomationRpcGateway/Private/Handlers/UI/UiHandler.cpp`. Added an anon-namespace `ResolveRuntimeWidgetWorld()` / `CollectRuntimeUserWidgets()` pair that resolves the target world PIE-first via `McpActorUtils::ResolveQueryWorld(TEXT("auto"), ...)` (now `#include "Utils/ActorUtils.h"`), and rewrote `ui.set_widget_text` to enumerate ONLY the resolved world's UserWidgets, write to the matching named TextBlock, echo the mutated `owning_user_widget` in the response, and return `WIDGET_NOT_FOUND` instead of silently succeeding on an editor-world decoy when no live instance matches. Audited and fixed the two siblings the ticket flagged: `ui.set_widget_image` and `ui.set_widget_visibility` previously used `TObjectIterator` + first-`GetWorld()` (non-deterministic instance) and now resolve PIE-first the same way (visibility matches the top-level UserWidget name or a named child; both echo `owning_user_widget`). Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/UI/TestUiSetWidgetTextResolvedWorld.cpp` (`EditorAutomationRpcGateway.ui.set_widget_text.HitsResolvedWorldInstance`) — spawns a live `UUserWidget` with a named `UTextBlock` in the resolved (no-PIE → editor) world, drives the production `ui.set_widget_text` handler, and asserts the text actually landed, the response echoes `owning_user_widget` (the new field is the revert detector — old code never set it), and that an unknown key returns `WIDGET_NOT_FOUND` rather than a silent success.
- `#1-initial-repro` `OPEN` reporter — Replayed live via mcp__editor-automation__call on WBP_PlayerHUD: after `editor.play` + clearing the auto-spawned HUD + a single `ui.create_hud` (`WBP_PlayerHUD_C_1`), `ui.set_widget_text {key:"InteractionText", value:"SCORE: 12500"}` returned success `{"key":"InteractionText","value":"SCORE: 12500"}`, but `widget.describe capture_source=live` on the painted `WBP_PlayerHUD_C_1` read back `"Text": ""`. Reproduced with PIE PAUSED (set again → success; live read → still `"Text": ""`), ruling out a gameplay-Tick reset. Root cause in `Handlers/UI/UiHandler.cpp` `ui.set_widget_text`: candidate list is built `GEditor->GetEditorWorldContext().World()` FIRST then the PIE viewport world, and the loop `break`s on the first matching TextBlock, so an editor-world (non-painted) instance intercepts the write while the painted PIE instance the screenshot renders is left empty. Same editor-world-first family as the fixed `B-inspect-misses-pie-world`, but in `UiHandler.cpp`, untouched by that fix.
