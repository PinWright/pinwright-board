---
id: E-pause-step-frame-does-not-freeze-umg
title: "editor.pause / step_frame promise deterministic stepping but do not freeze UMG — short-lived HUD animations still cannot be captured"
status: OPEN
severity: Medium
category: ergonomic
tags: [editor, pause, step_frame, umg, slate, hud, animation, capture, visual-review, docs]
---

# `editor.pause` + `step_frame` do not freeze UMG, so a 210 ms HUD animation is still uncapturable

`editor.step_frame` is documented as "Advance the active PIE session by exactly one frame and pause
again. **Useful for deterministic stepping during debugging.**" That promise reads as covering the
whole frame, and it is the obvious tool for capturing a short-lived UMG animation — a hit marker, a
damage indicator, a pickup pop — that is too brief to catch by polling `editor.screenshot*`.

It does not work for UMG, because pausing PIE stops the **world** tick while Slate/UMG keep ticking
on real time. A `UUserWidget::NativeTick`-driven animation therefore keeps running while the session
is "paused", and continues to run during the seconds of RPC round-trip between the call that
triggers it and the call that captures it.

## Repro (measured 2026-09-03T00:28Z, EAContentExamples58)

Target: `/Game/FPS/UI/WBP_HUD`'s hit marker, whose whole life is ~210 ms (alpha holds 80 ms then
eases out over 130 ms), driven from the widget's `Tick`.

1. `editor.play`, then `editor.pause` → `{"state":"paused"}`.
2. `object.call_function` `DebugHit` on the possessed pawn → fires the dispatcher, sets `HitAlpha=1`.
3. `editor.step_frame` → `{"success":true,"message":"Stepped one frame"}`.
4. `editor.screenshot_window` → PNG.

The marker is **absent** from the PNG. Measured: in an 80x80 px box centred on the viewport centre
the maximum channel value is 219 — the scene's bright panel — with **no** 255 HUD white anywhere.
Worse, the same paused frames show the **crosshair missing entirely** (0 pixels above 90 in that box)
although the compass and ammo text are still painted, because `UpdateCrosshair` had run once more
with a stale/zero spread and collapsed `CrosshairRoot`, while the text blocks simply kept their last
painted strings. A frame captured from the same session **after `editor.resume`** shows the crosshair
correctly (solid 255 arms at y 595-604 / 624-632).

So the paused capture is not merely empty, it is *misleading*: it shows a HUD in a state the running
game never renders, and a reviewer could easily file "the crosshair does not render" from it.

## What is wanted

Not necessarily a real UMG freeze — just don't let the docs imply one:

1. Say on `editor.pause` / `editor.step_frame` that the pause is **world-only** and that Slate/UMG
   continue to tick on real time, so widget-driven animation is not frozen or stepped, and
   widget state sampled while paused may not correspond to any rendered running frame.
2. Point at the thing that would actually work for this task. Nothing currently does: capturing a
   sub-frame-accurate UMG state needs either a Slate-tick freeze, or a capture verb that composites
   at a caller-chosen moment, or a "trigger and capture in one round trip" form.

## Workaround

None found. The animation was left unverified in two consecutive review rounds — once by the builder
polling `editor.screenshot` and missing the window, once by this critic using pause + `step_frame`.

## History
- `#1-filed` `OPEN` reporter — Filed after the pause+step route produced frames with no hit marker and
  a spuriously collapsed crosshair, while a resumed frame from the same session rendered correctly.
  The `step_frame` doc line "useful for deterministic stepping" is what led me to the approach.
