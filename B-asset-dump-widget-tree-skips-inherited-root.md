---
id: B-asset-dump-widget-tree-skips-inherited-root
title: "asset.dump emits empty placeholder tree.xml on child WBPs that inherit RootWidget from a parent"
status: DONE
severity: High
category: bug
tags: [asset-dump, widget, tree-xml, inheritance]
---

# asset.dump emits empty placeholder tree.xml on child WBPs that inherit RootWidget from a parent

Child WidgetBlueprints that don't override RootWidget locally produce a `tree.xml` containing only the comment `<!-- empty: WidgetTree has no RootWidget on this WBP -->`. The actual UI structure lives in the parent's dump, but consumers have no pointer back to it. 25+ such files observed in a 30k-asset sweep.

This breaks any consumer that walks the dump cache to render or audit a child widget — the tree appears empty when in fact the inherited parent provides the entire layout.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/UI/Hud/W_OnScreenJoystick_Left/tree.xml`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/Elements/Buttons/W_TutorialNextButton/tree.xml`.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/Elements/W_DemoNavigationHelper/tree.xml`.
4. Observe: only the `<!-- empty: WidgetTree has no RootWidget on this WBP -->` comment is emitted.

**Fix (proposed):** `WidgetXmlExporter::BuildWidgetTreeXmlWithDiagnostic` (Source/EditorAutomationRpcGateway/Private/..., lines 358–364) returns "empty by design" when the leaf WBP has no local RootWidget. Walk up `parentClass` and emit the inherited tree (with a marker noting the source ancestor), or at minimum embed an `inherited_from` pointer.

## History
- `#1-initial-repro` `OPEN` reporter — Child WBPs that inherit RootWidget from a parent get a `tree.xml` containing only the placeholder comment. Repro: `Game/UI/Hud/W_OnScreenJoystick_Left/tree.xml`, `App/App/UI/LobbyAndMenu/Elements/Buttons/W_TutorialNextButton/tree.xml`, `App/App/UI/LobbyAndMenu/Elements/W_DemoNavigationHelper/tree.xml`. Observed value: `<!-- empty: WidgetTree has no RootWidget on this WBP -->` and nothing else. 25+ such files in a 30k-asset sweep.
- `#2-walk-inherited-widget-tree` `IN-REVIEW` developer — `BuildWidgetTreeXmlWithDiagnostic` now consults `UWidgetBlueprintGeneratedClass::FindWidgetTreeOwningClass()` when the leaf WBP has no local RootWidget; emits the inherited tree with `inherited_from` attribute and a top-level XML comment naming the parent. Native-parent / no-BPGC paths still report `bEmptyByDesign=true` with refined Reason. Added `FWidgetXmlExporterInheritedRootTest` with both inherited-root and native-parent counterfactual cases.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on `/Game/UI/Hud/W_OnScreenJoystick_Left` and `/App/App/UI/LobbyAndMenu/Elements/Buttons/W_TutorialNextButton`. Both `tree.xml` files now lead with `<!-- inherited from /Game/UI/Hud/W_OnScreenJoystick_Right.W_OnScreenJoystick_Right -->` (resp. `W_ImageButton`) followed by the full inherited tree, with `inherited_from="..."` attribute on the root element.
