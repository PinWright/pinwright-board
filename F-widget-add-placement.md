---
id: F-widget-add-placement
title: "`widget.add` needs ordered placement"
status: DONE
severity: Medium
category: feature
tags: [widget-tree, add, placement, ordering, ergonomics]
---

# `widget.add` needs ordered placement

`widget.add` always appends new widgets to the end of the parent panel. There is
no way to insert at a specific sibling position. UMG draw / focus / layout order
in `VerticalBox`, `HorizontalBox`, and overlay panels is sibling-order
sensitive, so this gap forces a two-call dance for any non-tail insertion.

`F-widget-reparent-placement` (DONE) already added a `placement` object
(`{before|after|index}`) to `widget.reparent_widget` and extracted
`ResolveWidgetInsertIndex` as a shared helper used by both
`widget.reparent_widget` and `widget.duplicate`. `widget.add` should accept the
same `placement` arg and reuse the same helper.

## Proposed Fix

Add an optional `placement` param to `widget.add`. In
`WidgetAuthoringHelpers::AttachToParentOrRoot` (or a sibling overload), resolve
the insert index with `ResolveWidgetInsertIndex` and switch from
`TargetPanel->AddChild(ChildWidget)` to `TargetPanel->InsertChildAt(Index, ChildWidget)`.
Omitting `placement` keeps the current append behavior. Return `insertIndex`
in the response like `widget.reparent_widget` does.

## Workaround

Call `widget.add` then `widget.reparent_widget` with the same parent and
`placement:{...}`. Works, but it's two RPCs plus a redundant detach/reattach.

## History

- `#1-initial-request` `OPEN` reporter — Hit while assembling a `Card` VerticalBox: needed `ReasonText` between `PhotoPreview` and the Accept/Reject buttons, but `widget.add` only appends, and `bp.widget_tree` python access is blocked. Confirmed source: `WidgetAddHandler.cpp` calls `AttachToParentOrRoot`, which ends in `TargetPanel->AddChild(ChildWidget)` (WidgetAuthoringUtils.cpp:824). Shared placement helper `ResolveWidgetInsertIndex` already exists and is used by `widget.reparent_widget` and `widget.duplicate`.
- `#2-placement-via-attach-helper` `IN-REVIEW` developer — Added optional `placement` param to `widget.add`. Extended `AttachToParentOrRoot` (WidgetAuthoringUtils.h/.cpp) to take `Placement`, `OutInsertIndex`, and `OutErrorCode`; it now calls `ResolveWidgetInsertIndex` and uses `TargetPanel->InsertChildAt(Index, ChildWidget)`. Handler propagates `INVALID_PLACEMENT` (and finer NOT_FOUND/INVALID_PARENT) error codes from the helper, and returns `insertIndex` in the response.
- `#3-verify-fix` `DONE` tester — Verified: `widget.add?` schema exposes `placement` (object). Created temp WidgetBP, added a `VBox`, then appended Alpha, Gamma; inserted Beta with `placement:{before:"Gamma"}` (insertIndex=1), First with `placement:{index:0}` (insertIndex=0), AfterAlpha with `placement:{after:"Alpha"}` (insertIndex=2). `widget.describe` confirms final child order [First, Alpha, AfterAlpha, Beta, Gamma], matching expected. All three placement modes work, `insertIndex` returned per response. Temp asset deleted.
