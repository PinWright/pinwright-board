---
id: E-input-state-blind-to-x-pointer-grab
title: "drive.input_state cannot see an X11 pointer grab held by the editor itself; while it lasts every os_input/XTEST click silently vanishes, and on a shared :0 the grab confines every user's pointer"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [drive, drive.input_state, os-input, xtest, x11, linux, pointer-grab, shared-display, pie]
encounters: 2
costly: 1
lastSeen: 2026-09-30T12:30:00Z
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
- `#2-inverse-capture-without-grab` `OPEN` reporter - Second encounter, the inverse case: UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin `adb239fd`, editor PID 937718 on a shared `:0`. After an os_input click into a racing PIE viewport, `drive.input_state` reported `viewport_has_mouse_capture:true`, `mouse_capture_mode:"CapturePermanently"`, `mouse_lock_mode:"LockOnCapture"`. A hand-compiled `XGrabPointer` probe returned `GrabSuccess`, so no X grab was held (another app had X focus). So `drive.input_state` cannot tell an engine-side capture flag from a real OS grab in either direction, and on a shared display I again had to compile a probe to check that I was not confining other sessions' pointers. Workaround cost: a gcc build and a probe after each click batch. Not costly.
- `#3-grab-probe-and-refusal` `IN-REVIEW` developer - `drive.input_state` gains `os_grab: {pointer, keyboard, held_by_this_editor}` (null without X). PinWright's own X connection probes with `XGrabPointer`/`XGrabKeyboard` on the root and releases at once (`AlreadyGrabbed`/`GrabFrozen` = held by another client). X cannot name the holder, so `held_by_this_editor` is SDL's view instead (`SDL_GetGrabbedWindow`, or relative mouse mode on `SDL_GetKeyboardFocus`'s window, which SDL3 X11 implements as a confining `owner_events=False` pointer grab, the exact grab in `#1`), resolved by `dlsym(RTLD_DEFAULT)`, null when the entry points are missing. `drive.click`/`drive.hover` with `os_input` probe under the display lock before any motion and refuse with the new `POINTER_GRABBED` (payload `held_by_this_editor`), moving and pressing nothing. The message points at Shift+F1 or Slate when the editor holds the grab, and at `XF86LogGrabInfo` otherwise. The wiki documents the Shift+F1 release and the shared-display confinement. Pure LockOnCapture confinement (SDL_SetWindowMouseRect = XFixes barriers) holds no X grab and is unaffected. Files: `Handlers/Drive/DriveOsInput.{h,cpp}`, `Handlers/Drive/DriveInputStateHandler.cpp`, `Handlers/Drive/DriveActionHandlers.cpp` (os_input param doc), `Handlers/ErrorCodes.h` (`ERR_POINTER_GRABBED`), `docs/wiki-src/drive.md`, `Tests/Drive/TestDriveOsInput.cpp`. Test: `PinWright.drive.os_input.PointerGrabRefuses` (a stand-in X client grabs pointer+keyboard; asserts probe + `drive.input_state.os_grab` + `POINTER_GRABBED` with the pointer unmoved, then free again; skips `no-x-display` / `x-grab-already-held`).
