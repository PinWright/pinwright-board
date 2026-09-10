---
id: B-drive-observe-collapsed-ancestor-reads-visible
title: "drive.observe reports `visible:true` with non-zero geometry for children of a Collapsed parent — `Element.bVisible` is the widget's own `GetVisibility()` flag and the geometry is a stale cached paint value, and `drive.expect widget_visible`, the action gate and the settle fingerprint all inherit the lie"
status: OPEN
severity: High
category: bug
tags: [drive, drive.observe, drive.expect, widget_visible, visibility, collapsed, ancestor-propagation, cached-geometry, stale-geometry, slate, false-positive, pie]
encounters: 1
lastSeen: 2026-09-10T00:00:00Z
---

# A widget under a collapsed parent reports itself visible, at last frame's coordinates

Observed 2026-09-10 driving a PIE session through the MCP against UE 5.8, host project
`X:\src\unreal\unreal-fpv-new`, plugin at `d5dfb11b`. `drive.observe` on the game surface
listed `VoicePipName` with `visible:true` and non-zero geometry while its parent pip
overlay had `Visibility == Collapsed` and was demonstrably not on screen.

## Cause: own flag, plus a geometry nothing invalidates

`Source/PinWright/Private/Handlers/Drive/DriveLiveResolver.cpp:335-344`

```cpp
// bVisible follows EVisibility::IsVisible() ("is drawn") rather than == Visible, so that
// SelfHitTestInvisible status text (a common case for non-clickable HUD labels) still
// reports visible for verification reads.
Element.bVisible = SlateWidget->GetVisibility().IsVisible();      // :338
Element.bEnabled = SlateWidget->IsEnabled();
Element.bFocused = SlateWidget->HasAnyUserFocus().IsSet();

const FGeometry& CachedGeometry = SlateWidget->GetCachedGeometry();  // :342
Element.AbsolutePosition = CachedGeometry.GetAbsolutePosition();
Element.AbsoluteSize     = CachedGeometry.GetAbsoluteSize();
```

Two independent defects on adjacent lines.

**`:338` reads only the widget's own attribute.** `SWidget::GetVisibility()` returns the
widget's own `EVisibility`; `EVisibility::IsVisible()` is a local bit test
(`(Value & VISPRIVATE_Visible) != 0`). Slate never writes an ancestor's collapsed state
down into children — propagation happens at paint/hit-test time through the
`bParentEnabled` / arrangement path, not on the child's attribute. `SWidget::IsVisible()`,
which folds the parent chain, is **never called** anywhere in the drive path: `IsVisible()`
under `Handlers/Drive/` hits only `SWindow::IsVisible()` in the window-enumeration filter
(`DriveEditorChrome.cpp:229`).

**`:342` reads a geometry from the last frame the widget was painted.**
`GetCachedGeometry()` returns `SWidget::PersistentState.AllottedGeometry`, written during
the last paint pass. Slate skips arranging and painting a Collapsed subtree entirely, so
the cached `FGeometry` from when the pip *was* visible survives untouched — which is why
the element reports real, plausible, and wrong coordinates rather than zeros. Nothing in
the drive path invalidates or re-measures it, and there is no `ArrangeChildren` anywhere
under `Handlers/Drive/`.

**The walk does not prune collapsed subtrees.** `WalkAndCollect`
(`DriveLiveResolver.cpp:352-370`, `DriveEditorChrome.cpp:162-178`) recurses into
`Children->GetChildAt(Index)` unconditionally with no visibility test at any level, so
the child is reached, emitted and stamped from its own never-changed flag.

Three parent walks do exist in this path and none consults visibility:
`BuildWidgetPath` (`DriveLiveResolver.cpp:282-303`, `DriveEditorChrome.cpp:414-436`,
path string), `ComputeHandleBaseKey` (`DriveLiveResolver.cpp:394-425`, handle chain), and
`FDriveGameInput::IsWidgetInFocusPath` (`DriveGameInput.cpp:478-495`, focus).

## Surfaces affected

- **Game — affected** (`DriveLiveResolver.cpp:338,342`).
- **Editor chrome — affected identically.** `DriveEditorChrome.cpp:143-152` is a byte-for-byte duplicate of the same five lines (`:146` visibility, `:150` cached geometry).
- **Web — not affected in practice.** `DriveWebBridge.cpp:788` (`vis()`) checks only the element's own computed `visibility`/`display`, so it shares the flaw in principle, but `add()` at `:789` drops any element whose `getBoundingClientRect()` is zero-sized, and a descendant of `display:none` always measures zero. The geometry guard the Slate path lacks is what saves it.

Serialization: `Source/PinWright/Private/Handlers/Drive/DriveJson.cpp:116`.

## Why this is High, not cosmetic

`bVisible` feeds three further consumers, so the false positive propagates into
assertions and actions rather than staying a display defect:

- **`drive.expect` / `drive.wait_for` `widget_visible`** — `DriveConditionEval.cpp:197`, `Result.bMet = Match->bVisible;`. An agent asserting "the pip is hidden now" gets a green on a false premise, which is the rubric's High class exactly: a result the caller trusts and builds on.
- **The action gate** — `DriveActionCommon.cpp:218`, `if (!Resolved.Element.bVisible || !Resolved.Element.bEnabled)`. A target under a collapsed ancestor passes the guard and is then clicked at stale coordinates, i.e. at whatever is now drawn there.
- **The settle fingerprint** — `DriveFingerprint.cpp:85` hashes only `bVisible` elements. A collapse-driven UI change therefore produces no fingerprint delta, so settle reports `no_change_within_budget` for a transition that did happen.

## Fix

`bVisible` must mean "would be drawn", which requires the ancestor chain. Either call
`SWidget::IsVisible()` (folds the parent chain) instead of `GetVisibility().IsVisible()`,
or prune in `WalkAndCollect` — a Collapsed/Hidden parent's subtree should not be emitted
as visible at all. Keep the `SelfHitTestInvisible` allowance the `:335-337` comment
documents; that is a separate and correct concern.

The stale geometry is a second fix, not the same one: a widget that is not being painted
has no current geometry, so the response should report that rather than last frame's
numbers — zeroed, or a `geometryStale` flag — otherwise a fixed `visible` still leaves the
action gate clicking at fabricated coordinates.

No unit test pins any of this. Every `bVisible` reference under
`Source/PinWright/Private/Tests/Drive/` (`TestDriveConditionEval.cpp`,
`TestDriveFingerprint.cpp`, `TestDriveSettleDriver.cpp`) sets the flag on a synthetic
`FDriveElement`; nothing exercises the live Slate derivation.

## Workaround

Do not trust `visible` or the reported geometry from `drive.observe`, and do not use
`drive.expect widget_visible` to assert a widget is hidden. Confirm against pixels
(`drive.screenshot` / `editor.screenshot`) or read the ancestor's `Visibility` property
directly by reflection.

## Related

- `B-drive-observe-commonui-disabled-reads-enabled` (OPEN, Medium) — the same bug class on the adjacent line: `Element.bEnabled = SlateWidget->IsEnabled()` at `DriveLiveResolver.cpp:339` reads the widget's own flag, which CommonUI's disable path never writes. Same root shape (own-widget flag mistaken for effective state), different flag; a fix for one should cover both.
- `B-drive-click-misses-pie-game-viewport` (OPEN, Medium) — clicks no-op in PIE while observe reports valid hit-test geometry. Stale cached geometry is a plausible shared cause.
- `B-screenshot-designer-not-runtime-faithful` (OPEN, Medium) — `widget.screenshot_designer` draws runtime `Collapsed` widgets; different verb, same conceptual gap.
- `E-set-widget-visibility-bool-loses-eslatevisibility` (IN-REVIEW, Medium) — the write-side counterpart.

## History
- `#1-collapsed-parent-child-reads-visible` `OPEN` reporter — Filed from a PIE session on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `d5dfb11b`. `drive.observe` on the game surface reported `VoicePipName` as `visible:true` with non-zero geometry while its parent pip overlay was `Collapsed` and not rendered. Verified in source: `Element.bVisible = SlateWidget->GetVisibility().IsVisible()` at `DriveLiveResolver.cpp:338` reads only the widget's own attribute (Slate propagates ancestor collapse at paint/arrange time, never onto the child's flag), `SWidget::IsVisible()` is called nowhere in the drive path, `WalkAndCollect` (`:352-370`) recurses without any visibility test, and `GetCachedGeometry()` at `:342` returns the last painted `FGeometry`, which a collapsed subtree never updates — hence real-looking wrong coordinates instead of zeros. Editor-chrome surface is a byte-for-byte duplicate (`DriveEditorChrome.cpp:146,150`); the web path escapes only because it drops zero-sized elements (`DriveWebBridge.cpp:789`). Confirmed unfixed at HEAD (`d5dfb11b`): the `bVisible` line is unchanged since `b3ddbe55` introduced it, and no commit in either file's history touches visibility derivation. Severity High rather than Medium because the same flag drives `drive.expect widget_visible` (`DriveConditionEval.cpp:197`), the action gate (`DriveActionCommon.cpp:218`) and the settle fingerprint (`DriveFingerprint.cpp:85`) — so the wrong value is asserted on, acted on, and silences a real change, which is the rubric's silent-wrong-data class. No fix attempted.
