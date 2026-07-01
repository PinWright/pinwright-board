---
id: B-widget-describe-live-no-root
title: "widget.describe capture_source=live without asset_path fails to find UMG-as-root"
status: DONE
severity: High
category: bug
tags: [widget, describe, live, pie, commonui, umg-root]
---

# widget.describe capture_source=live without asset_path fails to find UMG-as-root

`widget.describe` with `capture_source: "live"` and no `asset_path` is the only documented way to capture live PIE widget state. It is also the only way to inspect what is *actually* on screen, since the asset XML for a UWidgetBlueprint does not reveal which sub-widgets were instantiated/added by gameplay code.

Today the call returns:

```json
{ "format": "text", "content": "Widget: Unknown" }
```

regardless of what is on screen. The handler rejects `asset_path` when `capture_source=live` (`INVALID_PARAMETER: asset_path ... not accepted when capture_source=live`), so there is no way to point it at a specific live root either. Net effect: live UMG capture is unusable in projects that use Lyra/CommonUI's `PrimaryGameLayout` — i.e. PDS — because the handler cannot locate the UMG-as-root user widget under the layered widget stack.

**Reproducer:**

1. Start PIE in PDS (`editor.play` returns `alreadyPlaying: true`).
2. With the race-end leaderboard on screen, call:
   ```json
   { "path": "widget.describe", "args": { "capture_source": "live" } }
   ```
3. Result: `Widget: Unknown` — no tree, no widget names, no visibility states.
4. The UE Widget Reflector at the same moment shows the live tree rooted at `SConstraintCanvas [GameViewport]` → `SObjectWidget (W_OverallUILayout_C_0)` → `MainOverlay` → `GameLayer_Stack` → `W_HUD_Race_Main` → `W_HUD_WaitingLobby` → ... (screenshot confirms this).

So the live root *exists* (`W_OverallUILayout_C_0`), but the MCP handler is not locating it.

**Expected behavior** — match what the Widget Reflector does with "Pick Painted Widget" + "Show widgets only" / "UMG as root" toggled on:

- Walk the active PIE viewport's Slate tree, identify the topmost `SObjectWidget` (UMG user widget) under the game viewport, and dump that user widget's tree using the same shape `widget.describe` already produces for asset-side capture.
- Include nested user-widget instances (CommonUI layers, switcher slots, list entries) by recursing through `SObjectWidget` boundaries — the asset-side describe already handles nested UMG correctly; live should too.
- For multi-rooted UI (game viewport layer + menu layer + modal layer in Lyra), either return all live UMG roots in an array, or accept an optional `instance_name` / `root_index` to pick one. The current `instance_name` parameter on `resolve_geometry` is a precedent.

**Workaround:** none. Callers fall back to dumping each suspected asset via `widget.export_xml` and guessing which subtree is live — but that misses runtime-added children, runtime `Visibility` overrides, and bindings driven by C++ code.

**Fix:** The live-capture code already locates the correct UMG root (`SConstraintCanvas` whose first child is `SObjectWidget` matches Lyra/CommonUI's `AddToScreen` wrapper, verified via Widget Reflector 'UMG as root' on `W_OverallUILayout_C_0`); the failure is in the registered text formatter, which only understands the asset-shape JSON and collapses live-shape responses to `Widget: Unknown`. Taught `WidgetDescribeFormatter.cpp::FormatWidgetDescribe` to detect the live snapshot shape (`capture_source`/`root` top-level fields) and dispatch to a dedicated `FormatLiveSnapshot` walker that mirrors the asset path's output.

## History

- `#1-initial-repro` `OPEN` reporter — In PDS PIE with the race-end leaderboard on screen, `widget.describe { capture_source: "live" }` returns `Widget: Unknown` while Widget Reflector clearly shows `W_OverallUILayout_C_0` rooted at the game viewport with a deep CommonUI subtree (`W_HUD_Race_Main` → `W_HUD_WaitingLobby` → `W_RaceEndLeaderboardPlayer` etc.). The handler also rejects `asset_path` together with `capture_source=live` (`INVALID_PARAMETER`), so callers cannot target a specific class either. Live UMG capture is the only way to answer "is button X currently visible / collapsed", and right now that question is unanswerable through MCP for any Lyra/CommonUI-based project.
- `#2-text-formatter-live-shape` `IN-REVIEW` developer — Live capture already finds the right UMG root; the text-format collapse to "Widget: Unknown" was the registered `widget.describe` text formatter (`WidgetDescribeFormatter.cpp`) reading only asset-shape fields. Added a live-shape detection branch + `FormatLiveSnapshot`/`FormatLiveNode` walker that emits Source/Viewport/Widgets header lines and a `slate_type "<display_name>"` tree with slot/binding/delegate lines, mirroring the asset path. Added `FWidgetDescribeTextFormatterRendersLiveSnapshotTest` in `TestWidgetDescribeFormatter.cpp` covering an `FLiveUiSnapshotJsonWriter::Write` output through the registered formatter; counterfactual: reverting the formatter change collapses the output to "Widget: Unknown" with no widget names.
- `#3-verify-live-describe` `DONE` tester — Verified: ran `editor.play`, then `widget.describe` with `{"capture_source":"live"}` and observed formatted text beginning with `Live Widget Capture`, `Source: live`, `Widgets: 19`, and the live UMG root `SObjectWidget "W_OverallUILayout_C_0"` instead of `Widget: Unknown`; stopped PIE afterward with `editor.stop`.
