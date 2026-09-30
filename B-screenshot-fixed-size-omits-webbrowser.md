---
id: B-screenshot-fixed-size-omits-webbrowser
title: "editor.screenshot with width/height (fixedSizeScenePlusUmg) drops CEF SWebBrowser content: WebUI panels are missing from the PNG with no warning"
status: OPEN
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
