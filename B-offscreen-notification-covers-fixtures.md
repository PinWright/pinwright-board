---
id: B-offscreen-notification-covers-fixtures
title: "Under -RenderOffscreen (640x360 virtual display) the editor's untitled topmost notification window covers pointer-test fixture windows; drive.click_occlusion.UncoveredTargetIsClicked skips with an uninformative \"host window ''\""
status: IN-REVIEW
severity: Medium
category: bug
tags: [offscreen, linux, notification-window, drive, drive.click, simulate_input, target-occluded, test-skips, misleading-error]
encounters: 4
lastSeen: 2026-10-01T08:46:38Z
---

# Offscreen fixture windows covered by the notification window

Under `-RenderOffscreen` the editor's virtual display is 640x360. The run log shows
`systemresolution.resx="640"` and `resy="360"`. At that size the editor's notification window
covers a large part of the screen. That window is untitled, topmost, and holds the toasts
(`SNotificationBackground`). An ordinary fixture window placed near the middle of the screen is
under it in Slate's window order. Every pointer injection at the fixture's point then goes to the
notification window.

Measured in offscreen run `3acc89ff90fb40b58257bc3da6f99741`, using temporary debug logging that
has since been removed:
- The `editor.simulate_input.MouseClickHonestSuccess` fixture sat at (207,107) with size 226x146.
  The widget path leaf at its button center (320,180) was an `SImage`, not the fixture's `SButton`.
- The next click, at (400,300), had leaf `SNotificationBackground`.
- The toast window is titled `''`, as is that fixture. So the handler's "delivered to window ''"
  was accurate, and it looked like the fixture.

After the fixture was made `.IsTopmostWindow(true)` (run `6114a9af28da440689ba99d7e3191f3b`), the
leaf became `SButton`, the button captured and `OnClicked` fired.

`drive.click_occlusion.UncoveredTargetIsClicked` has the same cause. Its fixture is at (420,320),
360x200, which on a 640x360 display overlaps the notification area. It skips with
`fixture-window-stacked-under-host-window -- The uncovered fixture sits under host window ''`, in
runs 0a3aa810, 16964ca9, 3acc89ff and 6114a9af. The skip and the `TARGET_OCCLUDED` refusal are
correct. The occluder is real and topmost in Slate's order. But `''` does not tell the reader it is
the notification window. On offscreen hosts the test never exercises its positive half.

Suggested fix, not done here:
- Make the drive fixture window `.IsTopmostWindow(true)` and move it inside the 640x360 display, as
  the simulate_input fixture now is.
- Optionally, name an untitled occluder by its widget type in `TARGET_OCCLUDED` and in the
  simulate_input response, for example the leaf widget under the point.

## History
- `#1-filed-as-stale-cursor-window` `OPEN` developer — Filed while fixing `B-simulate-input-cef-click-noop`. It first blamed a stale platform window-under-cursor. Evidence: run 0a3aa810 automation.log lines 26927-26933 and 28655-28662.
- `#2-cause-is-notification-window` `OPEN` developer — The `#1` theory was wrong for offscreen. Debug logging in run 3acc89ff showed Slate's routing and the Slate-order top window at the point agreeing, on the notification window. That window is topmost and untitled, and covers fixtures on the 640x360 offscreen display. The window-under-cursor cache was not stale. Title and body rewritten to the measured cause. The simulate_input fixture is fixed (topmost). The drive fixture remains; see the suggested fix above.
- `#3-host-plugin-window-covers` `OPEN` developer - Seen again in run b2aea516 (automation.log 26824, 27744-27779): `drive.click_occlusion.UncoveredTargetIsClicked` and four `drive.weblive.*` tests skipped as `fixture-window-stacked-under-host-window`, this time naming a host plugin window ('ULTIMATE BLUEPRINT GENERATOR - ...'), not the notification window. The web fixture (`F-drive-web-action-parity` #4) now places its window clear of every visible window and makes it topmost; the Slate-surface fixture here still does neither.
- `#4-fixture-topmost-and-clear` `IN-REVIEW` developer - The `drive.click_occlusion` fixture (`Tests/Drive/TestDriveClickOcclusion.cpp`) now follows the `drive.weblive` pattern. Both fixture windows are `IsTopmostWindow(true)`; a later top-most window is above an earlier one, so the cover still covers the target. The target is placed right of every visible window (`GetAllVisibleWindowsOrdered`), so neither the notification toast nor a host plugin window holds its center, and `UncoveredTargetIsClicked` should now run its positive half on offscreen hosts. An untitled occluder is now named by type: `TARGET_OCCLUDED` adds `occluding_window_type` (for example `Notification`) and a `recovery` call (`DriveActionCommon.cpp`). Tests: `PinWright.drive.click_occlusion.UncoveredTargetIsClicked`, `PinWright.drive.click_occlusion.OccludedTargetIsRefused` and the new `PinWright.drive.click_occlusion.OsInputOwnWindowIsRefused`.
