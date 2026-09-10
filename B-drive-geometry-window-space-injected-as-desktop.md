---
id: B-drive-geometry-window-space-injected-as-desktop
title: "drive reports element geometry in window space and injects it as desktop pixels with no `SWindow::GetPositionInScreen()` term — correct only while the owning window's client origin is (0,0), and the `drive` wiki calls the same field \"absolute screen-space\""
status: OPEN
severity: High
category: bug
tags: [drive, drive.click, drive.hover, drive.observe, geometry, coordinate-space, window-space, desktop-space, get-position-in-screen, slate-injection, docs-mismatch, pie, pie-in-viewport]
encounters: 1
lastSeen: 2026-09-10T00:00:00Z
---

# One field, two coordinate spaces, and the docs pick the wrong one

Surfaced 2026-09-10 while diagnosing `B-drive-click-misses-pie-game-viewport` on UE 5.8,
host `X:\src\unreal\unreal-fpv-new`, plugin `d5dfb11b`, PIE running inside the
level-editor viewport. Drive's reported game-surface geometry sat about (+37,+254) px
from the same widgets in captured viewport pixels. That particular delta turned out to be
**expected** — it is the level viewport's origin inside the editor frame, and drive
reports window space — but chasing it exposed a real defect underneath.

## What is wrong

Element geometry is produced in **window** space:

- `Source/PinWright/Private/Handlers/Drive/DriveLiveResolver.cpp:342-344` — `Element.AbsolutePosition = SlateWidget->GetCachedGeometry().GetAbsolutePosition()`. No window-origin, DPI or viewport term.
- Slate roots cached geometry at `SWindow::GetWindowGeometryInWindow()` — `C:\UE_5.8\Engine\Source\Runtime\SlateCore\Private\Widgets\SWindow.cpp:2143` — so the origin is that window's client top-left, not the desktop.
- The hit-test grid is separately offset by `GetPositionInScreen()` (`SWindow.cpp:2128`), and `FSlateApplication::LocateWidgetInWindow` is fed the raw **screen** coordinate (`SlateApplication.cpp:1925-1947`, gated by `IsScreenspaceMouseWithin`).

It is then consumed as **desktop** pixels, unconverted:

- `Source/PinWright/Private/Handlers/Drive/DriveActionCommon.cpp:229` — `TargetCenter = Element.AbsolutePosition + Element.AbsoluteSize * 0.5;`, passed verbatim to the inject lambda.
- `Source/PinWright/Private/Handlers/Drive/DriveInput.cpp:125` — `Cursor->SetPosition(RoundToInt(ScreenPos.X), ...)`, desktop pixels; `:130-139` builds the `FPointerEvent` with `ScreenPos` as its screen-space position.
- `DriveInput.cpp:146` — `SlateApp.LocateWindowUnderMouse(ScreenPos, ...)`; `:234` / `:239` — `ProcessMouseButtonDownEvent` / `UpEvent` on the window that resolves.

Nowhere between `DriveLiveResolver.cpp:343` and `DriveInput.cpp:125/146/234` is
`SWindow::GetPositionInScreen()` added. The path is therefore correct **only** when the
owning window's client origin happens to be desktop (0,0) — a maximized main editor
window, which is the normal case and is exactly why this has not been caught.

The plugin's own code states the right contract and the wiki states the wrong one:

- `Source/PinWright/Private/Handlers/UI/WidgetGeometryResolver.h:40` — "Window-space position from `FGeometry::GetAbsolutePosition()`".
- `docs/wiki-src/drive.md` describes `geometry.absolute` as "absolute screen-space".

So a caller who reads the wiki and feeds `geometry.absolute` to an external, DPI-aware
`SendInput` is being told to do something that works by accident.

## Blast radius

- **Every `drive.click` / `drive.hover` / `drive.drag` / `drive.scroll` on a non-maximized or secondary window** aims at the wrong desktop point. There is no error: `LocateWindowUnderMouse` simply finds a different window (or none), the press is routed there, and `drive.click` reports `no_change_within_budget` because it discards the press-handled bool (`DriveInput.cpp:196-203`).
- **Editor-chrome surface has the same latent bug.** `DriveEditorChrome.cpp:150-151` takes element positions from cached geometry, while `:460-461` is the one place the plugin *does* use `GetWindowGeometryInScreen()` — for `drive.list_windows` window rects. So window rects and element rects in the same response are in different spaces.
- **External tooling.** `E-drive-wiki-os-input-injection-notes` records an external DPI-aware `SendInput` at `geometry.absolute` actuating widgets in a real PIE session at 2560x1440 @150% — consistent with a maximized window at (0,0), i.e. the accident holding.

## What it should do

Pick one space, state it, and convert at the boundary. Either publish desktop-space
positions (add the owning `SWindow`'s `GetPositionInScreen()` when building the element,
so external injectors and `drive.*` agree), or keep window space and add the term inside
`FDriveInput` before `SetPosition` / `LocateWindowUnderMouse` / `ProcessMouseButtonDownEvent`
— but not the current mix. Whichever is chosen, the response should name the space in a
field rather than leaving it to the wiki, and `docs/wiki-src/drive.md` must be corrected
either way; it is wrong today regardless of which fix lands.

A regression test needs no PIE: build a window at a non-zero desktop origin, resolve an
element, and assert the injected point matches the widget's screen rect.

## Not the same as

- `B-drive-click-misses-pie-game-viewport` (OPEN, Medium) — the symptom ticket this was found under. Its `#3` entry records that the observed (+37,+254) is window-space-as-designed and that Slate pointer **capture** on the PIE viewport (`SlateApplication.cpp:5335-5377`, which skips `LocateWindowUnderMouse` entirely when a capture exists) is the more likely blocker for that session. This ticket is the separate, systematic coordinate defect; fixing it will not by itself fix that one.
- `B-drive-observe-collapsed-ancestor-reads-visible` (OPEN, High) — the *staleness* of the same cached geometry, a different fault on the same read.
- `B-simulate-input-cef-click-noop` (IN-REVIEW, High) — the earlier malformed-injection fix (`e08ff518`), which routed `editor.simulate_input` through `FDriveInput` and therefore inherited this coordinate handling rather than introducing it.

## Workaround

Maximize the editor window (client origin at desktop (0,0)) before using `drive.click` /
`drive.hover` or feeding `geometry.absolute` to any external injector. There is no
in-band way to detect the wrong case — the injection reports success either way.

## History
- `#1-window-space-fed-to-desktop-injection` `OPEN` reporter — Filed 2026-09-10 out of the `B-drive-click-misses-pie-game-viewport` investigation on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `d5dfb11b`, PIE hosted in the level-editor viewport. The reported (+37,+254) offset against captured viewport pixels is **not** the defect — drive reports window space and that delta is the level viewport's origin inside the editor frame. The defect found underneath is that the same window-space value (`DriveLiveResolver.cpp:342-344`, rooted at `SWindow::GetWindowGeometryInWindow`, `SWindow.cpp:2143`) is carried unconverted through `DriveActionCommon.cpp:229` into desktop-space injection (`DriveInput.cpp:125` `SetPosition`, `:146` `LocateWindowUnderMouse`, `:234` `ProcessMouseButtonDownEvent`) with no `SWindow::GetPositionInScreen()` term anywhere — correct only while the owning window's client origin is (0,0). The plugin already documents the correct contract at `WidgetGeometryResolver.h:40` ("Window-space position") while `docs/wiki-src/drive.md` calls the field "absolute screen-space", so the shipped guidance is wrong today independently of the code fix. Editor-chrome carries the same mismatch internally: element rects from cached geometry at `DriveEditorChrome.cpp:150-151`, window rects from `GetWindowGeometryInScreen()` at `:460-461`. Confirmed unfixed at HEAD: `git log --grep` for coordinate / screen-space / mouse across the five touching files returns only `e08ff518`, which routed `simulate_input` through `FDriveInput` without touching coordinate handling. Severity High: the wrong value is silently acted on (a click lands somewhere else and still reports a benign settle outcome) and the docs actively direct callers into it, which is the rubric's silent-wrong-data class; it escapes notice only because the common window layout makes the missing term zero. No fix attempted, and no board ticket mentioned window-space vs desktop-space, `GetPositionInScreen`, or PIE-in-viewport hosting before this one.
