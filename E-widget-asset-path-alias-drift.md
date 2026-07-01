---
id: E-widget-asset-path-alias-drift
title: "widget.* namespace: asset-path slot uses widgetPath only; no assetPath alias on most handlers"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# widget.* namespace: asset-path slot lacks `assetPath` alias on most handlers

Sibling of `E-blueprint-param-name-path-vs-assetpath` (DONE) and
`E-widget-remove-widget-param-name` (DONE). Those fixed the instance-name
selector and the `blueprint.*` asset-path slot. The `widget.*` asset-path
slot was not part of either sweep and still drifts.

Across `widget.*`, the asset-path slot (path to the `UWidgetBlueprint`) is
called `widgetPath` everywhere — `widget.set`, `widget.add`,
`widget.remove_widget`, `widget.rename_widget`, `widget.reparent_widget`,
`widget.set_animation_*`, etc. None of those accept `assetPath`, `path`,
`blueprintPath`, or the snake variants. `widget.describe` is the lone
exception: its declared canonical is `asset_path` with aliases
`widgetPath` / `path` / `assetPath` (`WidgetDescribeHandler.cpp:160-167`).

**Repro:** `widget.remove_widget {assetPath:"/App/App/UI/LobbyAndMenu/Elements/W_AnalyzerControls", widgetName:"SwitchVisibleLapsButton"}`
→ `[UNKNOWN_PARAMS] Unknown parameter(s) for 'widget.remove_widget': [assetPath]. Valid parameters: [widgetPath, widgetName, name, slotName, widget_name, targetName, cleanupNewOrphans]`.
Retry with `widgetPath` succeeds. Same drift on `widget.set`, `widget.add`,
`widget.rename_widget`, etc.

Source check: `WidgetHierarchyHandler.cpp:136-145` declares
`RPC_PARAM_REQ("widgetPath", ...)` with no asset-path aliases.
`WidgetSetHandler.cpp:107-118` same. The recent `widget.*` param-name
sweep that landed in `E-widget-remove-widget-param-name #3/#4` only
covered the *instance-name* selector (`widgetName` + aliases), not the
asset-path slot.

**Fix:** Mirror the blueprint-path resolver approach. Annotate every
`widget.*` handler's asset-path param with the shared path aliases
(`assetPath`, `path`, `blueprintPath`, `blueprint_path`, optionally
`requestedPath`) via dispatcher `FParamSpec` alias metadata — same
mechanism already used by `blueprint.*` after
`E-blueprint-param-name-path-vs-assetpath #4`. Either canonical name
(`widgetPath` on most handlers, `asset_path` on `widget.describe`) is
fine; standardize the alias set, not the canonical.

## History
- `#2-implemented` `IN-REVIEW` worker — Added shared `WidgetHandlerUtils.h` metadata helpers for canonical `widgetPath` aliases: `assetPath`, `path`, `blueprintPath`, `blueprint_path`, and `requestedPath`. Wired listed widget asset handlers to use the shared ParamSpec alias set and body-side alias lookup; kept `widget.describe` / `widget.export_xml` live-mode rejection semantics for asset-path fields. Added dispatcher metadata coverage for `widget.remove_widget`.
- `#1-filed` `OPEN` reporter — Hit `widget.remove_widget {assetPath:..., widgetName:...}` → `UNKNOWN_PARAMS [assetPath]`. Source: `WidgetHierarchyHandler.cpp:136-145` accepts `widgetPath` only; sibling `WidgetSetHandler.cpp:107-118` and the `widget.set_animation_*` family in `WidgetAnimationHandler.cpp` also accept `widgetPath` only. `WidgetDescribeHandler.cpp:160-167` is the lone exception that aliases `assetPath`/`path`/`widgetPath`. Parallel to `E-blueprint-param-name-path-vs-assetpath` (DONE) on the `blueprint.*` namespace; the dispatcher-level alias machinery from that ticket's `#4` is the reusable solution.
- `#3-verify-fix` `DONE` tester — Verified aliases live across the widget asset-path slot using `W_AnalyzerControls` from the repro. `widget.remove_widget {assetPath:...}` → `[NOT_FOUND] Widget '__McpVerifyNonexistent__' not found` (alias accepted; asset loaded). Also confirmed: `widget.set {assetPath:...}` → `NOT_FOUND` (widget), `widget.rename_widget {path:...}` → `NOT_FOUND` (widget), `widget.reparent_widget {blueprintPath:...}` → `MISSING_REQUIRED_PARAM newParent` (alias accepted), `widget.set_animation_loop {blueprint_path:...}` → `ANIMATION_NOT_FOUND`, `widget.add {requestedPath:...}` → `MISSING_REQUIRED_PARAM type`. Zero `UNKNOWN_PARAMS` on any alias. Wiki page `call("widget")` documents the shared aliases at the namespace level.
