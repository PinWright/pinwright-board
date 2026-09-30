---
id: B-editor-screenshot-world-echo-editor-map
title: "editor.screenshot during PIE reports world = the editor's map (L_Core) while the pixels come from the PIE world on another map"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, editor.screenshot, pie, world, response-metadata, wrong-data]
encounters: 2
lastSeen: 2026-09-30T12:30:00Z
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
- `#2-re-rated` `OPEN` triage — Severity Medium -> High. Silent wrong data on a normal path: the `world` field of every PIE `editor.screenshot` names the editor map rather than the rendered PIE world, and a caller using it to confirm what the capture shows gets a false answer; PIE game-viewport screenshots are routine, so no reach modifier.
- `#3-world-follows-capture-branch` `IN-REVIEW` developer — Root cause: `editor.screenshot` is a mutating (tick-unsafe) verb, so `UPinWrightSubsystem::DecorateAutomationResponse` adds `world` via `PinWrightWorldPrecondition::AddWorldField` -> `ResolveTargetWorldForMethod`, which returned `GEditor->GetEditorWorldContext().World()` for every non-`ui.*`/non-Niagara verb, while the handler captures `GEngine->GameViewport` whenever it exists (`ViewportHandler.cpp` editor.screenshot). Fix: `ResolveTargetWorldForMethod` now resolves `editor.screenshot` through the new `SelectScreenshotWorld(bHasGameViewport, GameViewportWorld, EditorWorld)` (`Dispatch/WorldPrecondition.{h,cpp}`) using the same `GEngine->GameViewport` branch test, so `world` (and `expectWorld` validation for this verb) names the game/PIE viewport's world on the game branch and the editor world otherwise. Test: `PinWright.world.precondition.ScreenshotReportsCapturedWorld` (`Tests/World/TestWorldPrecondition.cpp`). Wiki: `docs/wiki-src/editor.md` editor.screenshot. Live PIE-after-travel repro not re-run.
- `#4-second-hit-and-console-error-echo` `IN-REVIEW` reporter - Second encounter, pre-fix build: UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin `adb239fd` (no `SelectScreenshotWorld`). PIE started on `L_Core`, then travelled by `App.Launch` to `ShowcaseBaked` and to `L_PDS_Stadium`. About 12 `editor.screenshot` calls (`captureSource:"gameViewport"`, pixels from the Stadium PIE world) all returned `"world":"/Game/System/FrontEnd/Maps/L_Core.L_Core"`. Same resolution on another verb: `editor.console_command {command:"viewmode lit", world:"server"}` failed `EXEC_FAILED`, and its error payload echoed `{"world":"/Game/System/FrontEnd/Maps/L_Core.L_Core"}` although the command targeted the PIE server world. The #3 fix only covers `editor.screenshot`, so the console_command error echo may need the same treatment. Cheap.
