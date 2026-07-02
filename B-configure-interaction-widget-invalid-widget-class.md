---
id: B-configure-interaction-widget-invalid-widget-class
title: "interaction.configure_interaction_widget creates an InteractionWidgetClass variable with an unresolvable class type ('Invalid property ... replaced with Object' compile warning)"
status: OPEN
severity: Medium
category: bug
tags: [interaction, configure-interaction-widget, invalid-class-property, compile-warning, blueprint]
encounters: 1
lastSeen: 2026-07-02T09:14:38.1349323+03:00
---

# `interaction.configure_interaction_widget` creates a malformed `InteractionWidgetClass` variable

`interaction.configure_interaction_widget` adds an `InteractionWidgetClass`
member variable to the target Blueprint, but the variable's class/pin type does
not resolve to a valid widget class. On the next compile (here the following
`blueprint.add_interface` call) the engine logs — twice —
`Invalid property 'InteractionWidgetClass' class, replaced with Object. Please
fix or remove.` and substitutes `UObject` for the intended type. The chest
compiles `UpToDateWithWarnings`, and the widget-class slot is now the wrong type
(a bare `Object` reference), so it cannot hold the `UUserWidget` subclass it is
meant to reference.

This is **distinct** from the value-no-op defect the same verb also exhibits
(the `configured:true`-but-no-CDO-writeback family, folded into
`B-configure-chest-properties-no-cdo-writeback`). Here the problem is the *type*
of the created variable, not its default value: the var is structurally
malformed at creation, independent of whether the CDO write lands.

## Evidence

Struggle-audit of the `BP_TreasureChest` interaction build (namespace
interaction, 29 RPCs). `interaction.configure_interaction_widget
{showOnHover:true, showPromptText:true, promptTextFormat:"Press E to open"}`
returned `{"widgetClass":"","showOnHover":true,"showPromptText":true,
"promptTextFormat":"Press E to open","configured":true}`. The subsequent
`blueprint.add_interface` compile surfaced (twice)
`Invalid property 'InteractionWidgetClass' class, replaced with Object.` — the
widget-class variable this handler created has no resolvable class.

## Fix (proposed)

When `configure_interaction_widget` creates `InteractionWidgetClass`, give it a
valid class-reference pin type (`TSubclassOf<UUserWidget>` — `PC_Class` with
`PinSubCategoryObject = UUserWidget::StaticClass()`, or an object ref to a
concrete widget class) so the Blueprint compiles clean. Verify no
`Invalid property ... replaced with Object` warning remains after compile.

severity rationale: impact=creates a structurally-malformed member (wrong/degraded variable type) but WITH a visible compile warning (not silent) and the asset still compiles (soft degradation) × reach=interaction namespace, not every-session (no bump) -> Medium.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of `BP_TreasureChest`. `configure_interaction_widget {showOnHover:true, showPromptText:true, promptTextFormat:"Press E to open"}` returned `configured:true`; the next `blueprint.add_interface` compile emitted twice `Invalid property 'InteractionWidgetClass' class, replaced with Object. Please fix or remove.` — the widget-class variable the handler creates has an unresolvable class type and the engine downgrades it to `Object`. Asset compiled `UpToDateWithWarnings`. Distinct from the value-no-op family (`B-configure-chest-properties-no-cdo-writeback`) — this is a malformed variable TYPE at creation, not a dropped default value. Proposed: create `InteractionWidgetClass` as `TSubclassOf<UUserWidget>` so the BP compiles clean.
