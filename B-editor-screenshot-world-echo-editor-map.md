---
id: B-editor-screenshot-world-echo-editor-map
title: "editor.screenshot during PIE reports world = the editor's map (L_Core) while the pixels come from the PIE world on another map"
status: OPEN
severity: Medium
category: bug
tags: [editor, editor.screenshot, pie, world, response-metadata, wrong-data]
encounters: 1
lastSeen: 2026-09-29T13:00:00Z
---

# Screenshot response names the wrong world during PIE

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `8fcc0b2a`. PIE was started from the editor
map `/Game/System/FrontEnd/Maps/L_Core`, then travelled (`App.Launch mapeditor`) to `L_PDS_Stadium`.
`actor.find_by_class {world:"pie"}` reported `worldPath: /Game/Maps/UEDPIE_0_L_PDS_Stadium.L_PDS_Stadium`.
Every `editor.screenshot` in that session returned `captureSource: "gameViewport"` and a PNG of the
Stadium map-editor HUD, but its response said `"world": "/Game/System/FrontEnd/Maps/L_Core.L_Core"`.
That is the editor world, which was neither rendered nor active in PIE.

**Expected:** when the capture comes from the game/PIE viewport, `world` is the PIE world that was
rendered (here `UEDPIE_0_L_PDS_Stadium`), or the field is omitted. A caller that uses `world` to confirm
what the capture shows gets the wrong answer.

**Related:** `B-console-command-world-precondition-ensure`. The world-precondition decoration resolves
`L_Core` there too, with the same PIE layout, so this is probably the same resolution of the editor
world instead of the PIE world.

## History
- `#1-pie-travelled-map-wrong-world` `OPEN` reporter - Seen on about 8 screenshots in three editor sessions while reproducing a map-editor selection bug. The pixels were right and only the metadata was wrong, so no work was lost. Cheap.
