---
id: B-asset-dump-anim-skips-root-and-slot-bindings
title: "asset.dump widget_animations.json silently skips root- and slot-bound bindings (v1 binding shape)"
status: DONE
severity: High
category: bug
tags: [asset-dump, widget, animation]
---

# asset.dump widget_animations.json silently skips root- and slot-bound bindings (v1 binding shape)

When a widget animation binding has `bIsRootWidget=true` or `SlotWidgetName != None`, the entire binding is omitted from `widget_animations.json`. This drops most hover/press feedback animations on buttons. 16 files contain `uses an unsupported v1 binding shape` warnings.

Names skipped in observed dumps: `OnHover`, `OnHovered`, `OnHoveredWithResize`, `OnHoveredWithScale`, `OnPressed`, `Play`, `ShowToastForAWhile`, `FadeIn_Anim`, `FadeOut_Anim`. These are exactly the animations a UI consumer wants to inspect.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/Elements/Buttons/W_AnimatedButton/widget_animations.json`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/UI/Foundation/Buttons/W_LyraButton/widget_animations.json`.
3. Observe: `OnHover`, `OnPressed`, etc. are absent; warning notes "unsupported v1 binding shape".

**Fix (proposed):** `WidgetAnimationJsonSerializer.cpp:2358-2363` — `if (WidgetBinding && (!WidgetBinding->SlotWidgetName.IsNone() || WidgetBinding->bIsRootWidget)) continue;` — v1 binding shape was deliberately skipped pending a v2 design. Given how universal these bindings are, the skip is too aggressive. Either implement v1 support or downgrade to a per-binding warning preserving the rest of the data.

## History
- `#1-initial-repro` `OPEN` reporter — `widget_animations.json` silently drops every binding where `bIsRootWidget=true` or `SlotWidgetName != None`. Sample paths: `App/App/UI/LobbyAndMenu/Elements/Buttons/W_AnimatedButton/widget_animations.json`, `Game/UI/Foundation/Buttons/W_LyraButton/widget_animations.json`. Names dropped: `OnHover`, `OnHovered`, `OnHoveredWithResize`, `OnHoveredWithScale`, `OnPressed`, `Play`, `ShowToastForAWhile`, `FadeIn_Anim`, `FadeOut_Anim`. 16 files with the v1-binding-shape warning across one slice.
- `#2-emit-root-and-slot-bindings` `IN-REVIEW` developer — Replaced the v1-binding-skip in `WidgetAnimationJsonSerializer.cpp:~2358` with conditional emission of `isRootWidget` (bool) and `slotWidgetName` (string) on the existing BindingObj. JSON shape stays symmetric with the importer at line 2191. Added `FWidgetAnimationJsonExportRootAndSlotBindingsEmittedTest` covering root/slot/normal binding cases.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on `/App/App/UI/LobbyAndMenu/Elements/Buttons/W_AnimatedButton`. Fresh `widget_animations.json` now contains both `OnHovered` and `OnPressed`; root-widget binding emits `"isRootWidget": true` with full 2dTransform track data alongside the normal `InternalRootButtonBase` binding. No "unsupported v1 binding shape" warning string in output.
