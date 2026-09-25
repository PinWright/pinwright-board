---
id: E-widget-add-invalid-placement-names-no-parent
title: "widget.add INVALID_PLACEMENT `'before' widget 'X' is not a child of the target parent` names neither the resolved parent (root when parentName is omitted) nor X's actual parent"
status: OPEN
severity: Low
category: ergonomic
tags: [widget, widget.add, placement, invalid-placement, error-message, parentName, root]
encounters: 1
lastSeen: 2026-09-25T09:00:00Z
---

# The placement error hides which parent was used

## Symptom

`widget.add {widgetPath:"/App/App/UI/LobbyAndMenu/W_LyraFrontEnd", type:"W_LobbyLoginButton_C",
name:"W_LobbyLoginButton", placement:{before:"W_AppUserPanel"}}` returned
`[INVALID_PLACEMENT] Placement 'before' widget 'W_AppUserPanel' is not a child of the target parent`.
The docs say `parentName` "uses root if omitted"; the root is the Overlay `Root`, and `W_AppUserPanel` lives in
`CanvasPanel_151`. The message named neither, so it read as if `W_AppUserPanel` did not exist. Adding
`parentName:"CanvasPanel_151"` fixed it.

## Expected

The error should name the resolved parent (`Root`, and that it was defaulted) and the anchor's actual parent
(`CanvasPanel_151`) so the fix is one retry. Alternatively, when `parentName` is omitted and placement names an
anchor, default the parent to the anchor's parent.

## History
- `#1-placement-error-no-parent-name` `OPEN` reporter - Hit while restoring W_LobbyLoginButton into W_LyraFrontEnd (PDS, UE 5.8); cost one guess-and-retry.
