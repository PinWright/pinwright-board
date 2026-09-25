---
id: B-widget-add-type-rejects-full-class-path
title: "widget.add `type` (declared classref) rejects a full Blueprint class path `/App/.../W_X.W_X_C` with INVALID_CLASS, while the short `W_X_C` works"
status: OPEN
severity: Low
category: bug
tags: [widget, widget.add, classref, class-resolution, blueprint-class, invalid-class, param-type]
encounters: 1
lastSeen: 2026-09-25T09:00:00Z
---

# widget.add cannot take a full class path

## Symptom

`widget.add {widgetPath:"/App/App/UI/LobbyAndMenu/W_LyraFrontEnd",
type:"/App/App/UI/LobbyAndMenu/Elements/W_LobbyLoginButton.W_LobbyLoginButton_C", name:"W_LobbyLoginButton"}`
failed with `[INVALID_CLASS] Widget class not found: /App/App/UI/LobbyAndMenu/Elements/W_LobbyLoginButton.W_LobbyLoginButton_C`.
The same call with `type:"W_LobbyLoginButton_C"` resolved the class. The asset exists and was loaded.

## Expected

`type` is declared `classref`. Other `classref` params (`actor.add_component.componentType`,
`actor.spawn.classPath`, `animation.authoring.add_graph_node.nodeClass`) document and accept a full path. A full
object path is the unambiguous form (two Blueprints can share a short name), so `widget.add` should accept it, and
the asset path without the `_C` suffix as well.

## Workaround

Pass the short generated class name (`W_X_C`).

## History
- `#1-full-path-invalid-class` `OPEN` reporter - Hit while restoring W_LobbyLoginButton into W_LyraFrontEnd (PDS, UE 5.8). Not covered by `E-widget-add-type-class-discovery`, which is about discovering native class names.
