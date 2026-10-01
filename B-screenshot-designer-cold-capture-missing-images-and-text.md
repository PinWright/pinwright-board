---
id: B-screenshot-designer-cold-capture-missing-images-and-text
title: "widget.screenshot_designer target:preview on a cold widget returns success with images and text missing that a second capture of the same widget draws"
status: OPEN
severity: Medium
category: bug
tags: [widget, screenshot-designer, umg, cold-frame, texture-streaming, false-evidence]
encounters: 1
lastSeen: 2026-10-01T11:43:00Z
---

# A cold Designer preview capture is incomplete but reports the same success shape as a complete one

UE 5.8, Linux, wt2 checkout, offscreen editor. `widget.screenshot_designer {widgetPath:
/App/App/UI/LobbyAndMenu/W_LyraFrontEnd, max_size: 1400}` as the first capture of that widget in the
session returned success, 137510 bytes, `alphaZeroFraction` 0.671. The same call about 40 s later
(Designer closed in between, nothing edited) returned 251059 bytes, `alphaZeroFraction` 0.591. The
first PNG is missing the top-left logo image (PDS / GEOSCAN), the weekly-tournament panel's
background image, its title text, its "tournament is being prepared" text and the countdown digits.
The second has all of them. A third capture with the Designer already open matched the second
(251037 bytes).

Both responses have the same fields, so the caller cannot tell an incomplete frame from a complete
one. It looks like the cold-frame class closed for thumbnails in `B-thumbnail-cold-first-frame-no-stats`:
textures and font faces that are still streaming or lazily loading when the single `FWidgetRenderer`
draw runs. Not yet root-caused.

**Workaround:** capture twice and keep the second, or `editor.open_asset` and let the Designer tick
before capturing.

## History
- `#1-found-during-hang-repro` `OPEN` developer — Found while trying to reproduce
  `B-screenshot-designer-hangs-game-thread` (`#5` there). Files:
  `Saved/Screenshots/WidgetDesigner/repro_frontend_cold.png` (incomplete) and `repro_par_c.png`
  (complete) in the wt2 host checkout.
