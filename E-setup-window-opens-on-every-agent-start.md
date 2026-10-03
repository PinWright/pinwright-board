---
id: E-setup-window-opens-on-every-agent-start
title: "The 'PinWright Setup' window opens on every editor started by editor_start / editor_restart, over the PIE viewport, so the first drive.type / drive.click into PIE fails TARGET_OCCLUDED until an agent closes it by hand"
status: OPEN
severity: Low
category: ergonomic
tags: [setup-window, editor_start, editor_restart, pie, drive, occlusion, target-occluded, startup]
encounters: 2
lastSeen: 2026-10-02T22:11:00Z
---

# The setup window keeps reappearing in agent-started editors

## Symptom

UE 5.8, PDS (`unreal-fpv`), Linux. In this session the editor was restarted eight times through `editor_start` /
`editor_restart`, each time on an already configured project. A floating `PinWright Setup` window (about 1006x606,
centred) was open after the start. The first `drive.type {handle:"LastNameBox"}` into the PIE login form failed:
`[TARGET_OCCLUDED] Element 'LastNameBox' is covered at (1066, 806) by window 'PinWright Setup'`. `wmctrl -c
'PinWright Setup'` fixed it. The occlusion check itself worked and the error named the window, which made the fix
quick; the problem is the window reopening at all.

## Expected

Don't open the setup window when the project is already set up, or at least not in an editor that `editor_start` /
`editor_restart` launched for an agent (an automation start). If it must open, open it docked or behind the level
editor, not over the viewport.

## Workaround

Close it after every start (`wmctrl -c 'PinWright Setup'`, or a drive.click on its close button).

## History
- `#1-reopens-on-agent-start` `OPEN` reporter - Hit during PIE verification of the acceptance service pupil (PDS QA #1044). Related: `B-drive-chrome-click-hits-stacked-window` mentions the same window in a stacked-window click.
- `#2-occludes-own-x-display-tests` `OPEN` tester - Also opens in a manual `-unattended -RenderOffscreen` automation run on `DISPLAY=:0` (no `editor_start`; `bShowSetupScreenOnLaunch` defaults true in `PinWrightSettings.cpp:96` and `OnMainFrameCreationFinished` invokes the tab with no unattended/automation check). It covers the PIE viewport at (389, 345), so PinWright's own X-display tests `drive.click_after_gamepad_key.ReturnsCommonInputToMouse`, `drive.hover_hold.PieUserWidgetStaysHovered`, `drive.type_focus.PieFocusTakenElsewhereIsRefused` and `drive.type_focus.PieSelectAllSurvivesType` skip (`pie_viewport_occluded` / `level-viewport-covered-by-host-window`, naming `PinWright Setup`) and never run their assertions. In the same run without `-ini:EditorPerProjectUserSettings:[BpGeneratorUltimate]:bNeverShowWelcomeDialog=True` the host plugin's `ULTIMATE BLUEPRINT GENERATOR` welcome popup sits on top and is named instead. Suggested: skip the launch-time open under `FApp::IsUnattended()` / `GIsAutomationTesting`, or have the drive PIE fixtures close or move host windows before asserting.
- `#3-unattended-launch-gate` `OPEN` developer - Partial fix. `FPinWrightModule::OnMainFrameCreationFinished` (`PinWrightModule.cpp`) no longer invokes the setup tab under `FApp::IsUnattended()`, so the automation suite (`suite_argv` always passes `-unattended`) and offscreen/headless `editor_start` editors never open it; a port conflict is still logged. Test `PinWright.infra.setup_screen.NotOpenedUnattended` (skips with `editor_not_unattended` when the `-unattended` switch is absent, read via `FParse::Param` because `FApp::IsUnattended()` is always true while automation runs, and with `setup_screen_launch_disabled` when `bShowSetupScreenOnLaunch` is off). CHANGELOG + `wiki-src/unattended.md` updated. NOT covered: a `visible` `editor_start` (no `-unattended`), which is the `#1` report; that editor is the user's own window, so whether to suppress it there (e.g. a `-PinWrightAgent` flag from the proxy) is left open. Independently, the drive PIE fixtures now hide this editor's own windows over the PIE viewport (`Tests/Drive/PieViewportClearance.h`). Code-only, compile-checked; not yet run.
