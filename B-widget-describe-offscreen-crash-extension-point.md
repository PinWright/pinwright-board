---
id: B-widget-describe-offscreen-crash-extension-point
title: "`widget.describe` with `resolve_geometry.source=designer` crashes when tree contains UUIExtensionPointWidget"
status: DONE
severity: Critical
category: bug
tags: [widget-describe, resolve-geometry, offscreen-render, extension-point, crash, lyra, commonui]
---

# `widget.describe` with `resolve_geometry.source=designer` crashes when tree contains UUIExtensionPointWidget

Calling `widget.describe` with `resolve_geometry={"source":"designer", "force_visible_for_measure":true, "viewport_size":{w,h}}` on a Lyra/CommonGame front-end widget whose tree contains a `UUIExtensionPointWidget` crashes the editor with a NULL-`this` access violation on `UCommonLocalPlayer::CallAndRegister_OnPlayerStateSet`.

## Stack signature

```
UCommonLocalPlayer::CallAndRegister_OnPlayerStateSet  CommonLocalPlayer.cpp:36
UUIExtensionPointWidget::RebuildWidget                UIExtensionPointWidget.cpp:38
UWidget::TakeWidget                                   Widget.cpp:973
... (nested Overlay / SafeZone / CanvasPanel rebuilds)
ResolveViaOffscreen                                   WidgetGeometryResolver.cpp:447
FWidgetGeometryResolver::Resolve                      WidgetGeometryResolver.cpp:581
AutoHandler_22_                                       WidgetDescribeHandler.cpp:254
FRpcDispatcher::ProcessRequest                        RpcDispatcher.cpp:172
```

Exception: `0xc0000005 Access violation reading location 0x00000000`, `this = {UCommonLocalPlayer* const} NULL`.

## Root cause

`ResolveViaOffscreen` at `WidgetGeometryResolver.cpp:447` force-builds the widget tree to measure it. `UUIExtensionPointWidget::RebuildWidget` (Lyra CommonGame line 38) calls `CallAndRegister_OnPlayerStateSet` on `UCommonLocalPlayer::Get()` — which returns null in the editor-time context because no PIE session is active. `UCommonLocalPlayer::CallAndRegister_OnPlayerStateSet` dereferences `this` unconditionally → crash.

`UUIExtensionPointWidget` only makes sense inside a live gameplay session — it hooks player state changes to react to world events. Rebuilding it offscreen in the editor is outside its contract.

## Repro (observed this session, `W_LyraFrontEnd`)

```
widget.describe({
  asset_path: "/App/App/UI/LobbyAndMenu/W_LyraFrontEnd",
  resolve_geometry: {
    source: "designer",
    viewport_size: {w: 1920, h: 1080},
    force_visible_for_measure: true
  }
})
```

Crashes the editor process. Fresh start required; MCP must reconnect.

## Impact

- One describe call tanks the editor.
- Earlier in the same session, `widget.describe resolve_geometry.source=designer` on `W_AppReplayEditor` worked — the difference is that editor widget's tree contains only plain user widgets, no `UUIExtensionPointWidget`. Any front-end widget under `/App/App/UI/LobbyAndMenu/` likely hits this path because `W_LyraHUDLayout` / `SafeZone` shells contain extension points.
- No warning surface before the crash — no hint that describing a front-end widget with geometry is unsafe.

## Workaround

Pass `resolve_geometry: {"source": "off"}` (or omit `resolve_geometry` entirely) when describing any front-end / in-menu widget. Use `widget.export_xml` instead when slot/layout values are needed — the XML path reads cached layout data and does not trigger offscreen rebuild.

## Proposal (as implemented)

The fix ships two coupled changes:

1. **API simplification.** `resolve_geometry.source` string-enum is removed. The param now accepts either a bool or an object. Null (omitted) and `false` disable geometry; `true` and object form enable it. Geometry is **off by default** — most `widget.describe` callers want tree structure, not pixel positions, and opting in guards against accidental offscreen rebuilds. The internal resolver always runs the Designer → Live → Offscreen waterfall; tier forcing is no longer exposed because every forced tier was a footgun (Designer/Live silently return empty when their preconditions aren't met; Offscreen can crash on extension-point widgets).

2. **Extension-point guard.** Before invoking `Root->TakeWidget()`, `ResolveViaOffscreen` calls `UWidgetTree::ForEachWidgetAndDescendants` on the root tree, which walks both the immediate widget tree and every nested `UUserWidget`'s `WidgetTree`. The earlier flat `ForEachWidget` walk missed nested cases like `W_LyraFrontEnd → W_FrontEndHUDLayout → IExtensionPointWidget` — the offscreen rebuild cascades into those subtrees, so the detector has to as well. When a `UUIExtensionPointWidget` is found anywhere in the rebuild graph, the resolver short-circuits with `UI_EXTENSION_POINT_REQUIRES_RUNTIME`. Detection still uses the class path string to avoid hard-linking against the `UIExtension` module.

Workaround retained: `widget.export_xml` still returns layout data for front-end widgets without triggering offscreen rebuild.

## History
- `#1-crash-on-extension-point-widget` `OPEN` reporter — Attempted `widget.describe` on `W_LyraFrontEnd` with `resolve_geometry.source="designer", viewport_size=1920x1080, force_visible_for_measure=true` to inspect `VerticalBox_260` layout before re-adding the `W_ReplayEditorButton`. Editor crashed. Stack trace pinned the access violation to `UCommonLocalPlayer::CallAndRegister_OnPlayerStateSet` called from `UUIExtensionPointWidget::RebuildWidget` under `ResolveViaOffscreen`. Identical describe call on `W_AppReplayEditor` (earlier in this same session) returned cleanly — the difference is the presence of `UUIExtensionPointWidget` in the front-end tree. Required editor restart.
- `#2-api-rework-guard-added` `IN-REVIEW` sprint — Reworked `resolve_geometry` API from string-enum (`auto|designer|live|offscreen|off`) to bool-or-object with off default. Added `UUIExtensionPointWidget` pre-scan in `ResolveViaOffscreen` returning `UI_EXTENSION_POINT_REQUIRES_RUNTIME`. Regression tests in `TestWidgetGeometryResolver.cpp`: `FWidgetGeometryResolverParseRequestDefaultOffTest`, `FWidgetGeometryResolverParseRequestBoolTrueTest`, `FWidgetGeometryResolverParseRequestBoolFalseTest`, `FWidgetGeometryResolverParseRequestObjectTest`, `FWidgetGeometryResolverExtensionPointDetectionTest`. Files touched: `WidgetGeometryResolver.{h,cpp}`, `WidgetDescribeHandler.cpp`, `WidgetXmlExportHandler.cpp`, `EditorAutomationRpcGatewayTests.Build.cs` (added UIExtension dep). Breaking change: callers passing `source: "designer" | "live" | "offscreen"` now fall through to Auto waterfall; `source: "off"` is ignored (pass `false` or omit the field instead).
- `#3-verified-partial-live-tests` `DONE` tester — Verified (partial live, rest via regression-test coverage): (1) default-off — `widget.describe asset_path=/Game/App/UI/Test/W_McpVerifyTemp` (no `resolve_geometry`) returns the tree cleanly, no geometry rebuild triggered; (2) object form — `widget.describe ... resolve_geometry={viewport_size:{w:1920,h:1080}, force_visible_for_measure:true}` on the same safe widget returns cleanly; (3) old string-enum compat — `widget.describe ... resolve_geometry={source:"designer", viewport_size:{w:1920,h:1080}}` still accepted without error on the safe widget. Live extension-point crash verification was not attempted because the only known test target in this project (`W_LyraFrontEnd`) is currently blocked by the separate LoadBlueprintAsset-reentrant-crash (see B-bpir-class-resolve-reentrant-crash re-opened this session), so loading it to test the guard would crash the editor from a different code path. Trusting `FWidgetGeometryResolverExtensionPointDetectionTest` for that branch. Note: the MCP client JSON schema still types `resolve_geometry` as `object` only — passing the bool `true`/`false` form fails client-side validation with `Expected object, received string`. Server accepts bool-or-object per the fix, but the MCP JSON schema needs updating to surface that to callers.
- `#4-regression-dump-research` `OPEN` reporter — Regression: live dump-parity research called `widget.describe` on `/App/App/UI/LobbyAndMenu/W_LyraFrontEnd` with geometry resolution while comparing fresh asset dumps. The editor crashed again through `UCommonLocalPlayer::CallAndRegister_OnPlayerStateSet` from `UUIExtensionPointWidget::RebuildWidget`, reached via `ResolveViaOffscreen` at `WidgetGeometryResolver.cpp:651` and `WidgetDescribeHandler.cpp:327`. Current dump evidence shows the path still contains `W_FrontEndHUDLayout` with `IExtensionPointWidget`, so the extension-point guard either missed this nested subtree or was bypassed by the active geometry path.
- `#5-recursion-fix` `IN-REVIEW` developer — Fixed `ContainsExtensionPointWidget` to use `UWidgetTree::ForEachWidgetAndDescendants` instead of `ForEachWidget`, so the guard descends into nested `UUserWidget` widget trees (e.g. `W_LyraFrontEnd → W_FrontEndHUDLayout → IExtensionPointWidget`). Added `FWidgetGeometryResolverExtensionPointDetectionNestedTest` exercising a parent tree containing a child UserWidget whose own `WidgetTree` holds the extension-point widget; the test calls production `ContainsExtensionPointWidget` and asserts true. Files: `WidgetGeometryResolver.{h,cpp}`, `TestWidgetGeometryResolver.cpp`.
- `#6-verify-nested-guard-live` `DONE` tester — Verified live: ran `widget.describe asset_path=/App/App/UI/LobbyAndMenu/W_LyraFrontEnd resolve_geometry={viewport_size:{w:1920,h:1080}, force_visible_for_measure:true}` — the exact call that crashed in `#1` and `#4`. Editor did not crash; call returned the 58-widget tree cleanly, including the nested `W_FrontEndHUDLayout_C` UserWidget that transitively contains the extension-point widget. Nested-tree guard fires as designed; offscreen rebuild is short-circuited instead of cascading into `UUIExtensionPointWidget::RebuildWidget`.
