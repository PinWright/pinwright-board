---
id: E-input-state-blind-to-x-pointer-grab
title: "drive.input_state cannot see an X11 pointer grab held by the editor itself; while it lasts every os_input/XTEST click silently vanishes, and on a shared :0 the grab confines every user's pointer"
status: OPEN
severity: Medium
category: ergonomic
tags: [drive, drive.input_state, os-input, xtest, x11, linux, pointer-grab, shared-display, pie]
encounters: 1
costly: 1
lastSeen: 2026-09-29T16:25:00Z
---

# An editor-held X pointer grab is invisible to drive.* and swallows real clicks

The setup was listen-server PIE (host in the level viewport, 2-3 client windows) on a PDS race lobby,
where the host HUD requests `CapturePermanently`. After a real (XTEST) click into the host viewport,
the editor process kept an **active core pointer grab** for minutes, even after `editor.stop` ended PIE.
Xorg `XF86LogGrabInfo` output:

```
Active grab 0x3400000 (core) on device 'Virtual core pointer' (2):
      client pid 3435620 /sdb-disk/UE_5.8/Engine/Binaries/Linux/UnrealEditor /sdb-disk/src/unreal/unreal-fpv-wt1/PDS.uproject ...
      owner-events false, kb 1 ptr 1, confine 340006c, cursor 0x0
```

(A tiny `XGrabPointer` probe returned `AlreadyGrabbed`.) For the whole time:

- `drive.input_state` looked healthy: `cursor_captor: null`, `viewport_has_mouse_capture: false`. The only
  hint was `os_cursor` frozen at one point (2559,849) while the real pointer was moved elsewhere.
- Every XTEST click on the editor went nowhere. Editor chrome menus did not open and UMG buttons did
  not fire, with no error from any verb.
- The grab confines **the display's** pointer to the editor window. On a shared `:0` that also
  hijacks other sessions' real or XTEST input.

What released it: a properly held Shift+F1 (`keydown Shift_L`, `keydown F1`, `keyup F1`,
`keyup Shift_L`) with X focus on the editor, or quitting the editor. A sloppy key sequence sent F1 on its
own, which switched the PIE game viewport to wireframe.

**Ask:**
- `drive.input_state` should report whether the X pointer or keyboard is actively grabbed and by which
  PID (for example from an `XGrabPointer` probe that is immediately released).
- `drive.click os_input:true` should refuse with a coded error (`POINTER_GRABBED`) instead of
  injecting into a grabbed display.
- The os_input wiki should document the Shift+F1 release and warn about the shared-display effect.

## History
- `#1-stuck-grab-wt1` `OPEN` reporter - UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`, PDS QA #744 follow-up. Found with a hand-compiled `XGrabPointer` probe plus `xdotool key XF86LogGrabInfo`. Recurred after most host-viewport clicks while the host was a pilot (CapturePermanently HUD). Costly: an editor restart, about 40 minutes of failed click attempts, and a period in which the shared display's pointer was confined to my window.
