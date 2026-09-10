---
id: B-drive-observe-collapsed-ancestor-reads-visible
title: "drive.observe reports `visible:true` with non-zero geometry for children of a Collapsed parent — `Element.bVisible` is the widget's own `GetVisibility()` flag and the geometry is a stale cached paint value, and `drive.expect widget_visible`, the action gate and the settle fingerprint all inherit the lie"
status: IN-REVIEW
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
- `#2-effective-visibility-and-zeroed-stale-geometry` `IN-REVIEW` developer — Extracted the five duplicated state/geometry lines out of both `MakeElement` producers into one shared helper, `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveElementFactory.{h,cpp}` (named namespace, unity/ODR-safe), used by `Handlers/Drive/DriveLiveResolver.cpp` and `Handlers/Drive/DriveEditorChrome.cpp`. `SWidget::IsVisible()` does not exist in UE 5.8 (checked `SlateCore/Public/Widgets/SWidget.h`), so effective visibility is folded DOWN the walk instead: `WalkAndCollect` in both files now threads a `DriveElementFactory::FAncestorState` whose `ForChildrenOf` applies `bAncestorsVisible && Widget->GetVisibility().IsVisible()` — the same recurrence the engine uses in `FSlateInvalidationWidgetVisibility` (`SlateCore/Public/FastUpdate/WidgetProxy.h:47-53`) — with no per-widget parent re-traversal. A collapsed/hidden subtree is still WALKED and still EMITTED (pruning it would silently change `widget_present` / `count` / `text_*` against widgets that do exist); every element under it now reports `visible:false`. The `SelfHitTestInvisible` allowance is preserved, since only Collapsed and Hidden have `IsVisible()` false. The stale geometry is fixed as a second, separate thing: a non-drawn element's rect is ZEROED and a new `FDriveElement::bGeometryStale` is set, serialized as `geometry.stale:true` (omitted when false) in `Handlers/Drive/DriveJson.cpp`, so the action gate can no longer aim at whatever now occupies last frame's coordinates. Downstream consumers inherit the fix unchanged: `DriveConditionEval.cpp:197`, `DriveActionCommon.cpp:218/229`, `DriveFingerprint.cpp:85`. `docs/wiki-src/drive.md` now documents `enabled`/`visible` as effective ancestor-folded state and the new `geometry.stale`. Regression tests in `Source/PinWright/Private/Tests/Drive/TestDriveElementFactory.cpp`: `PinWright.drive.element_factory.CollapsedAncestorHidesChild` drives the REAL editor-chrome walk over a fixture window holding a Collapsed border around a labelled text leaf, and `PinWright.drive.element_factory.AncestorStateFoldsVisibilityAndEnabled` pins the fold itself with no window or paint. Counterfactual: revert `FillStateAndGeometry` to `Element.bVisible = SlateWidget->GetVisibility().IsVisible()` and `CollapsedAncestorHidesChild`'s `TestFalse("a child of a Collapsed ancestor reads visible:false")` fails, because the leaf's own visibility is asserted (anti-vacuously) to still be Visible, so the false can only come from the ancestor fold; drop the zeroing and the same test's `AbsolutePosition.IsNearlyZero()` assertion fails.
- `#3-stale-geometry-consumer-gates` `IN-REVIEW` developer — Closing two holes the zeroing in `#2` opened, plus one honesty gap in its own contract. (a) `drive.drag`'s `to_handle` release point was ungated: `DriveActionHandlers.cpp` resolved the element and took `AbsolutePosition + AbsoluteSize * 0.5` with no actionability check at all, so a collapsed release target would have dragged to desktop **(0,0)** and still reported a benign settle outcome. The three-term rule is now one shared predicate, `FDriveActionCommon::IsActionable(const FDriveElement&)` (`Handlers/Drive/DriveActionCommon.{h,cpp}`, `bVisible && bEnabled && !bGeometryStale`), called by both the press-target gate in `RunAction` and the new release-target gate in `DriveActionHandlers.cpp`, which refuses with `TARGET_CHANGED`; the geometry term is kept explicit rather than folded into `bVisible` so the "never aim at a rect Slate did not measure this frame" invariant survives if staleness ever stops implying invisible. (b) `geometry_in_bounds` on a stale element compared a `(0,0)-(0,0)` box, which `DriveBoxContains` reports as CONTAINED by any `expected_bounds` whose min is at or below the origin — so the fix could newly return `met:true` for a collapsed widget. `DriveConditionEval.cpp` now short-circuits `bGeometryStale` to `met:false` with `actual:"stale"` and a detail naming why there is nothing to compare, ahead of the unset-bounds branch. (c) Audited every other `AbsolutePosition`/`AbsoluteSize` consumer under `Handlers/Drive/`: `DriveFingerprint.cpp:25-28` is stable either way (stale elements are excluded from `Compute` and compare as a stable zero rect in `Diff`), `DriveSetOfMarkLayout.cpp:33-34` drops the zero rect as too-small rather than marking a wrong point, `DriveJson.cpp` reports it with the `stale` flag, and `DriveListWindowsHandler` / `DriveEditorChrome:464` are window rects, not elements. No further gaps. (d) `DriveTypes.h` now states that `bGeometryStale` is derived as `!bVisible` and is therefore SUFFICIENT but not COMPLETE: it does not cover a visible widget that was never painted or was culled this frame, which would need a per-frame arrangement stamp Slate does not expose; and it records the one exception to desktop space, content under a retained `SRetainerWidget`/`URetainerBox` painted into an `SVirtualWindow` at (0,0), which `docs/wiki-src/drive.md` also now carries. Test additions in `Source/PinWright/Private/Tests/Drive/TestDriveElementFactory.cpp`: `CollapsedAncestorHidesChild` now asserts that the center an ungated caller would compute from the collapsed element is exactly desktop (0,0) and that `IsActionable` refuses it while accepting the live control element; `DisabledAncestorDisablesChild` and the pure `AncestorStateFoldsVisibilityAndEnabled` assert the gate too, so it is pinned even on a host that cannot paint. The fixture teardown now ticks Slate after `RequestDestroyWindow` (destruction is only queued), so the fixture window cannot still be enumerated by the next test's editor-chrome walk. Counterfactual for the drag gate: remove the `IsActionable` call from `DriveActionHandlers.cpp` and a collapsed `to_handle` is accepted, producing the (0,0) release point the test asserts is what an ungated computation yields; remove `!bGeometryStale` from `IsActionable` and `TestFalse("a stale element is refused by the action gate")` fails.
