---
id: B-screenshot-fixed-size-omits-webbrowser
title: "editor.screenshot with width/height (fixedSizeScenePlusUmg) drops CEF SWebBrowser content: WebUI panels are missing from the PNG with no warning"
status: IN-REVIEW
severity: Medium
category: bug
tags: [editor, editor.screenshot, fixed-size, cef, webbrowser, webui, pie, silent-wrong-data]
encounters: 1
lastSeen: 2026-09-30T12:24:00Z
---

# Fixed-size screenshots omit WebUI (CEF) widgets

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin `adb239fd`, standalone PIE with the PDS race
analyzer open: a UMG `W_HUD_Analyzer_WebView` hosting an `SWebBrowser` / `SWebBrowserView` (visible,
desktop 77,164 705x464 per `drive.observe`).

- `editor.screenshot {width:1920, height:1080}` gave `captureMode:"fixedSizeScenePlusUmg"`, `fixedSize:true`.
  The PNG showed the 3D scene and the engine on-screen messages only. The whole analyzer panel (pilot
  roster, timeline, playback controls, Back button) was missing.
- `editor.screenshot {}` a moment later gave `captureMode:"nativeBackBuffer"`, 704x464. The panel was there,
  including the roster rows the verification needed ("1 GENTLE FALCON --:--", "2 BEST LAP --:--").

Neither response said the web content was left out, so a caller comparing HUD state from the fixed-size
image would conclude the panel was not on screen.

**Expected:** the fixed-size composite includes `SWebBrowser` content, or the response carries a warning
(for example `omittedWidgets:["SWebBrowser"]`) and the wiki names the limitation next to the fixed-size
description, pointing to the native capture for WebUI panels.

**Cost:** cheap here (a second capture), but the first image was plainly wrong for the question asked.

## History
- `#1-analyzer-panel-missing` `OPEN` reporter - "Filed from the PDS QA #915 repro. The fixed-size PIE screenshot omitted the CEF race-analyzer panel; the native capture showed it."
- `#2-measured-omission-report` `IN-REVIEW` developer - The engine-side cause was not found by reading (SViewport -> MakeViewport with the CEF slate texture looks drawable by FWidgetRenderer; no PIE+CEF host was available to reproduce), so the fix MEASURES instead of assuming: after the off-screen game-layer render, `PinWrightScreenshotUtils::FindWebBrowsersMissingFromOverlay` (ScreenshotUtils.h/.cpp) walks the visible game-layer tree for `SWebBrowserView`, re-renders the layer with each one at render opacity 0, and reports the browser when nothing changes. `editor.screenshot` (ViewportHandler.cpp) now returns `omittedWidgets` (one `"SWebBrowserView"` per omitted instance, `[]` when none) on every fixed-size game capture, plus a `warning` pointing to the native capture when non-empty. CEF content is not composited by this change; the omission is reported. Wiki: editor.md (`### editor.screenshot`), visual-review.md. Test: `PinWright.editor.screenshot.FixedSize.WebBrowserContentIsCompositedOrReported` - real CEF browser in a fixture window with a magenta page, waits until the window back buffer shows it, then asserts magenta-in-layer XOR reported, and that a layer the browser contributed nothing to is reported (fails if the detector is removed). Its AddInfo line `off-screen layer drew the CEF page: yes|no` is the first evidence of whether FWidgetRenderer drops CEF at all; skips with the marker when CEF is unavailable or the page never paints.
