---
id: B-screenshot-designer-hangs-game-thread
title: "widget.screenshot_designer wedged the game thread for 10+ min on a C++-parented Widget Blueprint; the editor had to be killed and unsaved edits were lost"
status: OPEN
severity: Critical
category: bug
tags: [widget, screenshot-designer, umg, hang, game-thread, data-loss]
encounters: 1
costly: 1
lastSeen: 2026-09-24T01:25:00Z
---

# widget.screenshot_designer wedged the game thread on a C++-parented Widget Blueprint

UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, editor started with
`-RunningUnattendedScript`, no PIE. Sequence on `/App/App/UI/LobbyAndMenu/Elements/W_AppUserPanel`
(parent class is project C++ `UAppUserPanel`, a `UCommonUserWidget`):

1. `widget.add` x2 and `widget.set` x2 (new SizeBox + `W_ImageButton` child), then
   `blueprint.compile` (UpToDate, 0 errors) at 01:24:36Z. Package dirty, not saved.
2. `widget.screenshot_designer {widgetPath, filename, max_size: 1400, visibilityOverrides:
   {SB_SchoolLogin: Visible, LessonBlock: SelfHitTestInvisible, SB_Login: Collapsed,
   VB_Buttons: Collapsed}}`.

The call returned `stream read failed: timed out`. From then on the process was `Not Responding`
with about one core busy (CPU time 643 s to 1096 s over ~8 min), the frame counter stopped, and
nothing further was logged: not even `LogAssetEditorSubsystem: Opening Asset editor`, so the
hang sits before or inside the designer open. The gateway's I/O thread answered
`EDITOR_NOT_READY: game thread has not completed a tick for 128 s ... No PinWright RPC is in flight`,
which is wrong: the in-flight RPC was `widget.screenshot_designer`. The editor was killed by PID
after ~10 min; the unsaved widget edits were lost and had to be redone. Log preserved at
`Saved/Logs/PDS-hang-screenshot_designer-44296.log` in the host.

Control: after a restart, `editor.open_asset` on the same saved asset (and on
`W_LyraFrontEnd`, which contains it) opened the real UMG designer and the editor stayed responsive,
so a person opening the asset does not hit this; the hang is specific to the capture path.
Not the same as `B-geometry-offscreen-runs-native-construct` (that one asserts in
`NativeConstruct` via `resolve_geometry`; here there is no assert and the designer preview does not
run `NativeConstruct`), nor `B-screenshot-designer-leaves-designer-open-compile-crash`.

**Workaround:** do not use designer captures on this asset family; verify layout in PIE with
`editor.screenshot`. Save before any capture.
**Fix (proposed):** bound the designer-open/preview-resolve retry in
`ResolveDesignerPreviewTargetWithRetry` with a wall-clock limit that returns an error, and have
the stall reporter name the in-flight method.

## History
- `#1-hang-on-capture` `OPEN` reporter — Filed from the school-login UMG pass on
  `W_AppUserPanel`; costly (editor killed, edits redone, ~15 min lost).
