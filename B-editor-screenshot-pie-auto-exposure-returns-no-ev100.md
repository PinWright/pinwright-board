---
id: B-editor-screenshot-pie-auto-exposure-returns-no-ev100
title: "`editor.screenshot {exposure:{mode:\"auto\"}}` on the PIE path returns `viewCount:0` and no `ev100Equivalent`, so the documented pin-then-compare recipe cannot be followed"
status: DONE
severity: Medium
category: bug
tags: [capture, screenshot, exposure, pie, ev100]
---

# `editor.screenshot` auto-exposure on the PIE path reports no `ev100Equivalent`

The `editor.screenshot` wiki page gives an explicit recipe for making a set of captures comparable:

> To make N shots comparable at the exposure the scene resolves to: take one `{mode:"auto"}` shot,
> read `viewport.exposure.ev100Equivalent` from it, and pass THAT as `ev100` on the rest.

On the PIE / game-viewport path that value is never returned, so step one of the recipe cannot be
completed.

## Reproduction (this checkout, 2026-09-07, PIE running in `T_AI`)

    editor.screenshot {filename:"b10_expose_probe.png", width:1280, height:720,
                       exposure:{mode:"auto"}}
    -> {"captureSource":"gameViewport", "fixedSize":true,
        "exposure":{"mode":"auto","pinRequested":false,"pinned":false,
                    "restored":true,"viewCount":0}}

No `ev100Equivalent`, no `adapted`, and `viewCount: 0`. Repeated after `editor.eject` (so the world
had certainly rendered, and a `screenshot_window` grab of the same moment produced a normal lit
frame) — identical response, `viewCount` still 0. The PNG itself is written and is a real image.

`viewCount: 0` looks like the cause rather than a separate symptom: the exposure block is presumably
populated by walking the processed views, and on this path none are counted.

## Why it matters

The value is available elsewhere — `render.capture_open_level {exposure:{mode:"auto"}}` returns
`adapted 0.00208, ev100Equivalent 8.9` (quoted in `B-no-way-to-capture-pie-pixels` `#5`), and the
preview-capture path returns it too (see `E-capture-preview-pose-ignores-bounds-shape` `#3`). So the
recipe works everywhere except the one path that captures a running game.

Consequence in this project: with no measured EV to pin, I guessed one for a `PostProcessVolume`
(manual metering, EV100 1.5) and rendered the scene almost entirely black — a whole capture slot
spent, the level saved with a bad exposure, then reverted. The page warns against feeding `adapted`
back as `ev100`; it does not warn that on this path there is nothing to feed back at all.

## Ask

Populate `exposure.ev100Equivalent` (and `adapted`) on the game/PIE branch as the other capture
verbs do. If the value genuinely cannot be sampled there, say so in the response — a
`pinWarning`-style field, or `ev100Equivalent: null` with a reason — rather than omitting the key,
and add a line to the page saying the recipe does not apply on the PIE path. Silence reads as "the
scene has no exposure", which is not true.

## Workaround

None for the recipe. Either pin an arbitrary `ev100` and iterate by eye across several captures, or
capture through `editor.screenshot_window` with `ShowFlag.EyeAdaptation 0` and accept whatever fixed
EV that pins (which is what this stream is doing, and it blows out any frame containing a muzzle
flash).

## Notes

- Distinct from `F-one-verb-for-exposed-and-aimed-pie-capture`, which is about `editor.screenshot`
  being unable to aim at an actor while `screenshot_window` cannot expose. This one is narrower: the
  verb that *can* expose does not report the number its own documentation tells you to read.

## History

- `#1-auto-observes-drawn-view` `IN-REVIEW` developer — Still reproducible by code: the game branch only registered its view extension for a pin, and the native path only drew a frame for a pin, so `{mode:"auto"}` had nothing to count (`viewCount:0`) and no field to fill. `Utils/ScreenshotUtils.cpp`: `FGameViewportExposureViewExtension` is now also created for an explicit auto request (`FGameViewportCaptureOptions::bObserveExposure`) as an observe-only extension, the native path draws one frame whenever the extension is live, and `SetupView` (runs last) records the family's `ExposureSettings.bFixed/FixedEV100` and `FSceneView::GetLastEyeAdaptationExposure()` onto `FGameViewportCaptureMetadata`. `Handlers/Editor/ViewportHandler.cpp` publishes the level path's field contract on the top-level `exposure` block: `fixed`, `adaptedMeasured`, `adapted`, `ev100Equivalent`, `adaptedSource` (`readback` unpinned, `fixedPin` pinned); with no completed readback it says `adaptedMeasured:false` plus an `adaptedReadbackPending` reason instead of omitting the key. Omitted `exposure` is unchanged (no extra draw, no measurement). Wiki: `docs/wiki-src/editor.md`, `docs/wiki-src/render.capture-exposure.md`; CHANGELOG. Test: `PinWright.editor.screenshot.PieExposureAndAim` (owned host-neutral PIE; fixed-size auto, then fixed+aimed captures) and `PinWright.render.capture_open_level.PieWorldWarning` (pure text rule + source contract on the handler wiring), both in `Source/PinWright/Private/Tests/EditorOps/TestPieCaptureExposureAim.cpp`. Fails on revert: viewCount is 0 and `adaptedMeasured` is absent. If the PIE readback is still pending the measured-value half emits a skip marker. Syntax-checked with the clang -fsyntax-only fastcheck (UBT module flags), all four .cpp OK; no UHT-visible declarations changed; not built or run yet.
- `#2-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full: `PinWright.editor.screenshot.PieExposureAndAim` passed with 0 warnings. The `eye_adaptation_readback_pending` skip marker did not fire, so the measured half ran. On an owned host-neutral PIE session, `editor.screenshot {exposure:{mode:"auto"}}` took the `gameViewport` branch with `viewCount > 0` (the reported value was 0), `pinned:false`, `adaptedMeasured:true`, an `ev100Equivalent`, `adaptedSource:"readback"` and `adapted > 0`. That closes the ask: populate ev100Equivalent and adapted on the PIE branch. The pending case says `adaptedReadbackPending` instead of dropping the key; the test asserts that too, but that branch did not run here. `docs/wiki-src/editor.md` and `render.capture-exposure.md` were checked. Limit: Linux Vulkan offscreen only.
