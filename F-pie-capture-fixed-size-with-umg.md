---
id: F-pie-capture-fixed-size-with-umg
title: "No verb captures the PIE viewport at a caller-specified resolution, so UMG cannot be reviewed at 1080p/4K"
status: OPEN
severity: High
category: feature
tags: [ui, umg, screenshot, pie, resolution, dpi, hud, visual-review, editor, render]
---

# No fixed-size PIE capture, so a HUD can never be reviewed at its target resolutions

A HUD is only correct *at a resolution*. UMG scales through
`[/Script/Engine.UserInterfaceSettings]` `UIScaleCurve` keyed on the viewport's shortest side,
so font size, stroke width, sub-pixel snapping and title-safe margins all change with the
viewport. Reviewing a HUD therefore means capturing it at the shipped resolutions —
conventionally 1920x1080 and 3840x2160. **No verb can do this.**

- `editor.screenshot` — PIE branch captures at the **native** PIE viewport size only; there is
  no width/height parameter (its own wiki page says so and redirects fixed-size intent to
  `render.capture_open_level`). It does now composite the UMG overlay
  (`B-screenshot-omits-umg-overlay` `#2`), so it is the *only* verb that can see the HUD at all.
- `render.capture_open_level` — takes exact `width`/`height`, but renders the **Level Editor**
  viewport, which has no PIE world and no viewport-added UMG. Exact size, wrong surface.
- `misc.set_viewport_resolution` — a silent no-op (`B-set-viewport-resolution-noop`).
- `editor.resize_window` — resizes the **whole editor window's client area**, not the PIE
  viewport inside it. The viewport is the window minus docked-panel chrome, so a caller must
  reverse-engineer that overhead by trial and error, and the achievable viewport is capped by
  the desktop.

The two capabilities that exist (exact size; sees UMG) are in different verbs and cannot be
combined.

## What this cost

Measured 2026-09-02 on this machine (desktop 5120x1440). Editor window 1887x1210 produced a
1364x979 PIE viewport, i.e. ~523x231 of chrome. Reaching a **1920x1080** viewport needs a
~2443x1311 window — possible here. Reaching **3840x2160** needs a ~4363x2391 window: the
desktop is 1440 px tall, so it is **physically unreachable through window resizing on any
display shorter than ~2400 px**, which is most displays. The UI stream's build report records
the same dead end ("`editor.screenshot` returns the PIE viewport's native size (1364x979 here)
and `render.capture_open_level` cannot see UMG, so neither is reachable"), shipped all six
evidence frames at 1364x979, and had to downgrade its 4K claim to reasoning about the DPI curve
rather than a capture. A critic then could not check 4K either.

At 1364x979 the DPI scale is 0.906, so **every measurement taken from such a frame has to be
divided by 0.906 to mean anything**, and sub-pixel defects that only appear at scale 1.0 or 2.0
(a 5x5 crosshair dot landing on a half pixel on an even-width viewport) are invisible in the
evidence.

## What is wanted

Either of:

1. `editor.screenshot` gains optional `width`/`height` that render the PIE viewport offscreen at
   that size — applying the UIScaleCurve for the requested size, not the window's — and report
   the applied `dpiScale` alongside `width`/`height`.
2. A `render.capture_pie` sibling of `render.capture_open_level` with the same `width`/`height`
   contract, targeting the PIE game viewport and compositing the Slate/UMG overlay the way
   `CaptureGameViewportToPngFile` now does.

Either way the response should carry the **applied UMG DPI scale**, because that is the number
that makes a HUD measurement comparable across captures.

Distinct from its neighbours: `B-set-viewport-resolution-noop` is one method's no-op;
`E-fixed-size-capture-discovery` is a docs breadcrumb on the *level-editor* branch;
`B-screenshot-omits-umg-overlay` fixed *whether* the HUD appears, not *at what size*.
This ticket is the missing capability itself.

**Workaround used:** none for 4K. For measurement, capture at native size, read the viewport
height, evaluate `UIScaleCurve` at that height by hand, and divide every measured pixel figure
by the result.

## History

- `#1-filed` `OPEN` reporter — Filed while critiquing `/Game/FPS/UI/WBP_HUD` against Call of
  Duty reference stills, where the whole review turns on element size in pixels at 1080p and 4K.
  `editor.screenshot` gave 1364x979 and nothing else; `render.capture_open_level` takes a size
  but cannot see a viewport-added HUD; `editor.resize_window` only moves the outer window and
  cannot reach a 2160-tall viewport on a 1440-tall desktop. Every size figure in the review had
  to be back-scaled by the 0.906 DPI factor, and the 4K half of the brief could not be answered
  at all.
