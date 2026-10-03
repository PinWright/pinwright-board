---
id: E-drive-observe-image-buttons-unlabeled
title: "drive.observe labels image-only buttons as `SButton` (no tooltip, brush or item data), so catalog tiles cannot be told apart without clicking them"
status: OPEN
severity: Low
category: ergonomic
tags: [drive, drive.observe, label, image-button, tooltip, brush, list-view, tile-view, linux, accessibility]
encounters: 1
lastSeen: 2026-09-28T09:32:00Z
rice: [1, 2, 1, 1]
priority: 17
---

# Tiles with only an image have no identity in the element list

The PDS map-editor catalog (SKGMultiplayerLevelEditor) shows placeable items as image tiles: an `SButton` wrapping
an `SImage`, no text. In `drive.observe` output every tile is an `SButton` whose `label` is `SButton`, so the list
is N identical rows differing only by handle and rect. The only way to learn which tile is which prop was to
click it, place the item in the level and look at what appeared.

Cause, plugin `8fcc0b2a`: `ExtractLabel` (`Handlers/Drive/DriveLiveResolver.cpp:193-228`) tries accessible text,
then `STextBlock` text, then the widget tag, then the type name. Accessible text is compiled out on Linux
(`WITH_ACCESSIBILITY` is off; the comment at `:195-196` says so), and a button's own text lives in child
`STextBlock`s, which an image-only button does not have. Nothing reads the tooltip (`SWidget::GetToolTip()`), the
child `SImage` brush's resource object, or the owning `UUserWidget` / list-item object.

**Workaround:** click and place each candidate, or read the widget Blueprint and its data assets offline.

**Fix (proposed):** before falling back to the type name, use in order: the widget's tooltip text; for a button
with a single `SImage` child, that brush's resource object name (texture / material asset); for an entry of a
`UListView` / `UTileView`, the item object's name or class (the catalog's data asset). Surface the extra
identity as its own field (e.g. `asset` / `item`) if it should not go into `label`.

## History
- `#1-catalog-tiles-all-sbutton` `OPEN` reporter - UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`, PIE in PDS map-editor mode. Items could only be identified by placing them. Lost work not measured, so recorded as cheap.
