---
id: B-drive-observe-clipped-offscreen-reads-visible
title: "drive.observe reports visible:true for buttons of a closed dropdown that is translated out of a ClipToBounds parent; drive.click / drive.hover on them return no_change_within_budget and do nothing"
status: IN-REVIEW
severity: Medium
category: bug
tags: [drive, drive.observe, drive.click, drive.hover, visibility, clipping, render-transform, clip-to-bounds, false-positive, silent-noop, pie]
encounters: 1
lastSeen: 2026-09-25T09:00:00Z
---

# Elements clipped away by a render-translated panel read as visible and actionable

## Symptom

UE 5.8, PDS PIE front end, host `unreal-fpv`. `W_LobbyLoginButton` (header dropdown) keeps its menu in a
`Visible` VerticalBox inside a `ClipToBounds` parent; closed, the box sits at `RenderTransform Translation Y=-366`
and a hover animation slides it down. With the dropdown closed, `drive.observe {surface:"game",
instance_name:"W_OverallUILayout", interactables_only:true}` listed
`W_OverallUILayout_C_0/W_LyraFrontEnd_C_0/W_LobbyLoginButton/MyRecords/SCommonButton` (and `MyLessons`, `MyDrones`,
`ChangeAvatar`, ...) as `visible:true`, with geometry `{x:1646, y:-95, w:320, h:62}`, `{y:-32}`, `{y:31}`, which is
above or on top of the header, not on screen. `drive.click` on `MyDrones` (slate path, then `os_input:true`) and
`drive.hover` on it returned `outcome:"no_change_within_budget"`, `changed_count:0`, and the log showed no new
screen. Nothing in the response says the target was clipped. A real X11 mouse (xdotool) hovering the header opened
the dropdown and a click on the item then worked.

## Expected

Visibility should take the clip rect of ancestors with `ClipToBounds` into account (an element whose rect lies
fully outside its clipping ancestor's rect is not visible), or the element should carry a `clipped:true` flag. The
action gate should then refuse the click with `TARGET_CHANGED` / `TARGET_OCCLUDED`-style detail instead of a benign
settle outcome.

## Repro

1. PIE the front end, log in with a password account (dev header shows).
2. `drive.observe` the game surface: the dropdown buttons read `visible:true` with rects above the header.
3. `drive.click` `.../W_LobbyLoginButton/MyDrones/SCommonButton`: `no_change_within_budget`, no screen opens.

## Workaround

Hover the header with a real OS mouse (xdotool) to open the dropdown, then click the item by its desktop
coordinates.

## History
- `#1-closed-dropdown-reads-visible` `OPEN` reporter - Filed from the school-computer compatibility verification (W_LobbyLoginButton restored in W_LyraFrontEnd). Related but distinct: `B-drive-observe-collapsed-ancestor-reads-visible` covers Collapsed ancestors; here nothing is collapsed, the element is only clipped by a ClipToBounds ancestor after a render translation.
- `#2-ancestor-clip-rect-folded` `IN-REVIEW` developer - The shared element-state derivation (`DriveElementFactory`, used by both the game walk and the editor-chrome walk, hence observe / expect / click / hover / drag targeting) now threads a desktop-space clip rect down the walk: `ClipToBounds` / `ClipToBoundsAlways` intersect it, `ClipToBoundsWithoutIntersecting` replaces it (OnDemand ignored), and it is dropped below an `SRetainerWidget` (retainer-local geometry). An element (or ancestor) whose render bounding rect has no positive-area overlap with that clip reads `visible:false` with `geometry.stale` and zeroed rect, so the action gate refuses it with `TARGET_CHANGED` (visible=false). Partly clipped elements stay visible. Known ceiling: bounding rects, so a rotated clipper over-reports visibility; an element Slate culled keeps its last painted rect, which is what is compared. Files: `Source/PinWright/Private/Handlers/Drive/DriveElementFactory.{h,cpp}`, `DriveTypes.h` (field comments), `Tests/Drive/TestDriveElementFactory.cpp`, `docs/wiki-src/drive.md`, `CHANGELOG.md`. Tests: `PinWright.drive.element_factory.ClipRectRejectsRectsOutsideTheClipper` (pure), `PinWright.drive.element_factory.ClippedOutChildIsNotVisible` (live Slate fixture: 200x40 ClipToBounds box with leaves translated 0 / -10 / -60 px; inside and partial visible, outside visible:false + not actionable).
