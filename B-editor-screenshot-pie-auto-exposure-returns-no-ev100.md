---
id: B-editor-screenshot-pie-auto-exposure-returns-no-ev100
title: "`editor.screenshot {exposure:{mode:\"auto\"}}` on the PIE path returns `viewCount:0` and no `ev100Equivalent`, so the documented pin-then-compare recipe cannot be followed"
status: OPEN
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
