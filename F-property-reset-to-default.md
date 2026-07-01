---
id: F-property-reset-to-default
title: "Reset a UPROPERTY to its class default (clear override)"
status: DONE
severity: Medium
category: feature
tags: [property, widget, reflection, override, undo, ergonomics]
---

# Reset a UPROPERTY to its class default (clear override)

`property.set` and `widget.set` write values, but neither offers a "clear override / reset to class default" mode. UE's editor UI has this — the per-property revert arrow next to any modified field — but the MCP surface does not expose the underlying reset operation.

This bites any time an agent makes a transient property change (for screenshot prep, A/B comparison, or just exploration) and needs to clean up. Setting the property back to "the same value as the class default" looks correct in the live UObject but the asset still serializes the explicit override on save, polluting the diff.

## Repro

1. `widget.export_xml widgetName=HotKeys-Fly` — output contains no `Visibility` attribute (property is at class default `Visible`, no override).
2. `widget.set widgetName=HotKeys-Fly properties={Visibility:"Collapsed"}` then `widget.set widgetName=HotKeys-Fly properties={Visibility:"Visible"}`.
3. `widget.export_xml widgetName=HotKeys-Fly` — output now shows `Visibility="Visible"` even though the value matches the class default. The override flag persists.
4. `editor.save_all` will write the explicit override into the .uasset, changing the on-disk diff with no semantic change.

## Desired behavior

Add a reset method (or a `reset: true` flag on `property.set` / `widget.set`):

```json
{
  "path": "property.reset",
  "args": {
    "objectPath": "/App/.../W_HUD_DroneGameMenu.W_HUD_DroneGameMenu:WidgetTree.HotKeys-Fly",
    "propertyName": "Visibility"
  }
}
```

Behavior:
- Restore the property to its class default and clear the override flag so subsequent `export_xml` (with default include_defaults=false) omits it.
- Mark asset dirty (or accept `markDirty: false` to suppress).
- Return `oldValue`, `defaultValue`, `wasOverridden` so callers can confirm.

UE engine reference: `FProperty::IdenticalInContainer` is the check that drives delta serialization; the editor's per-property revert button uses `IDetailPropertyRow::ResetToDefault` / `FResetToDefaultPropertyContext`. For UWidget subobjects in a Widget Blueprint tree, the override state lives on the instance and the class default is on the BP CDO of the widget's class.

## Workaround

`python.execute` with manual reflection:

```python
import unreal
wbp = unreal.load_asset("/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu")
widget = wbp.widget_tree.find_widget(name="HotKeys-Fly")
default = widget.get_class().get_default_object()
widget.set_editor_property("visibility", default.get_editor_property("visibility"))
# Above sets the value but does NOT necessarily clear the override flag at the FProperty level.
```

This still leaves the override marker in some cases. The robust fix needs C++ (`FProperty::CopyCompleteValue` from CDO + clearing the per-instance modification record) which is hard to do correctly from Python alone. Hence the request for a first-class handler.

## History

- `#1-initial-request` `OPEN` reporter — During screenshot prep on `W_HUD_DroneGameMenu`, an agent set `Visibility=Collapsed` on three sibling overlays for a clean capture, then "reverted" them to `Visible` via `widget.set`. The widgets had no explicit `Visibility` override pre-edit; post-edit the asset now has `Visibility="Visible"` baked in even though that matches the class default. Save would persist a no-op diff. No `widget.*` or `property.*` handler resets to class default.
- `#2-property-reset-handler` `IN-REVIEW` developer — Added first-class `property.reset` in `UtilityPropertyHandler.cpp`, documented it in `docs/wiki/property.md`, and added `FPropertyResetClearsExplicitOverrideTest` to verify the handler copies the class default while clearing `FOverridableManager` explicit override state.
- `#3-verify-widget-visibility-reset` `DONE` tester — Verified: `widget.set` changed `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu:WidgetTree.HotKeys-Fly` `Visibility` to `Collapsed`, then `property.reset` on `Visibility` returned `oldValue: Collapsed`, `defaultValue: SelfHitTestInvisible`, `wasOverridden: true`, and `isOverridden: false`; follow-up compact `widget.export_xml` for `HotKeys-Fly` omitted `Visibility`.
