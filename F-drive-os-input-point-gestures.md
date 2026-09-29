---
id: F-drive-os-input-point-gestures
title: "No os_input pointer gesture at an arbitrary viewport point or world actor, and no hold/move-while-pressed control, so real-mouse tests of 3D viewport interaction fall back to raw xdotool with no safety gates"
status: OPEN
severity: Medium
category: feature
tags: [drive, os-input, xtest, drive.click, drive.drag, pie, game-viewport, world-to-screen, gestures]
encounters: 1
costly: 1
lastSeen: 2026-09-29T13:00:00Z
---

# Real-mouse gestures on 3D content need raw xdotool

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `8fcc0b2a`. The task was to reproduce a
map-editor bug where clicking placed objects does not select them, using faithful OS input. That meant:

- a click on a 3D actor in the PIE game viewport (a gate banner, a gizmo arrow);
- a gizmo drag and a box-select drag on the viewport;
- a click whose pointer moves 4-12 px while the button is held (an unsteady hand), a zero-hold click,
  and a right-button drag.

`drive.click` only accepts a `handle` from `drive.observe`, so it cannot target a viewport pixel or an
actor. It also has no hold-duration or move-while-pressed parameters. `drive.drag` has no `os_input`. So
I scripted `xdotool mousemove / mousedown / mouseup`. I mapped screenshot pixels to desktop pixels by
hand, by matching a UMG element's `geometry.absolute` against the screenshot, and I added my own
check that the X window under the pointer was my editor. Batching roughly 150 trials through a direct
HTTP helper (for `object.call_function GetSelection` readback) was the other half of the fallback.

**Ask:**
- `drive.click` / `drive.drag` accept a target point instead of a handle: game-viewport pixel
  coordinates, or `{actor: <path>, component?}` projected through the PIE player's view.
- `os_input` gains `button`, `hold_ms` and an optional `path` of intermediate points while pressed.
- `drive.drag` gains `os_input`.
- The os_input gates from `B-drive-click-os-input-foreign-x-window` and
  `B-drive-os-input-own-window-occlusion` apply to these gestures too.

## History
- `#1-map-editor-selection-repro` `OPEN` reporter - Filed from the PDS map-editor first-click selection repro (board `B-mapeditor-first-click-select` in the game repo). The whole real-mouse matrix ran on raw xdotool. Because xdotool has no occlusion gate, part of one batch went into a peer session's fullscreen game (see `B-drive-click-os-input-foreign-x-window` #3). Costly: about 1 h of scripting plus one lost batch.
