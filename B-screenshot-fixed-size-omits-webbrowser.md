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
- `#3-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.editor.screenshot.FixedSize.WebBrowserContentIsCompositedOrReported` passed in w23-final, but through its composited branch: the run logged `off-screen layer drew the CEF page: yes; reported omitted: 0`. In a fixture window the off-screen layer render does draw CEF, so the omission did not reproduce and the detector's positive path (a really omitted browser reported in `omittedWidgets` plus a `warning`) was not exercised. The #1 case (PIE game viewport with the UMG `W_HUD_Analyzer_WebView`, `editor.screenshot {width, height}`) was never run, and offscreen runs do not exercise PIE fixed-size game capture. Needs: the #1 repro in PIE with a WebBrowser HUD at a fixed size; the panel must be in the PNG or named in `omittedWidgets`.
