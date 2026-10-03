---
id: E-os-input-headless-editor-undocumented
title: "drive.click os_input:true cannot work in an editor started with editor_start visible:false (no DISPLAY, no X window), but neither editor_start nor drive.click says so up front; the error does not suggest the remedy"
status: OPEN
severity: Low
category: ergonomic
tags: [drive, drive.click, os-input, xtest, editor_start, visible-false, render-offscreen, headless, docs]
encounters: 1
costly: 1
lastSeen: 2026-09-29T12:55:00Z
rice: [2, 2, 1, 1]
priority: 33
---

# os_input and visible:false are incompatible, and only a failing call reveals it

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin clone `61c243f5`. I started the editor
with `editor_start {visible:false, extra_args:[-SKIPCOMPILE, ...]}`. The spawned command line was
`... -RenderOffScreen -unattended -RunningUnattendedScript ...` (PID 3217388). The process had no
`DISPLAY` in its environment and no X windows (`xdotool search --pid` empty). After PIE started,
`drive.click {handle:"AnonymousLoginButton", os_input:true}` returned
`[INVALID_ARGUMENT] XOpenDisplay(NULL) failed: this process has no X display (DISPLAY unset, or access denied).`

The refusal itself is correct: with `-RenderOffScreen` there is no window for XTEST to reach. The gap is
discoverability. `editor_start`'s `visible` parameter and the `drive.click` `os_input` notes each describe
their own side, but neither says that a windowless editor rules out os_input for the whole session. The
error also doesn't say what to do. A task that asks for both `visible:false` and faithful OS input finds
this out only after a boot and a PIE start, and then has to restart.

**Workaround:** `editor.quit`, then `editor_start {visible:true}` from a process with `DISPLAY=:0` (and
`XAUTHORITY`) in its environment. On a shared display, then see `B-drive-click-os-input-foreign-x-window`
and `B-drive-os-input-own-window-occlusion`.

**Fix:** say it in `editor_start`'s `visible` description and in the `drive.click` os_input notes. Have
the os_input `INVALID_ARGUMENT` detect `-RenderOffScreen` and say "this editor was started windowless
(`visible:false`); os_input needs an editor started with `visible:true` on an X display".

## History
- `#1-headless-os-input-restart` `OPEN` reporter - Filed during a PDS localization check in wt2. Costly: one editor restart (about 2 minutes of boot plus a PIE start) to switch from `visible:false` to `visible:true` so os_input could run.
