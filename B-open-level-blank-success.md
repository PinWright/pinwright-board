---
id: B-open-level-blank-success
title: "render.capture_open_level silently succeeds with a uniform black PNG"
status: OPEN
severity: High
category: bug
tags: [render, capture_open_level, viewport, screenshot, blank, diagnostics, silent-success]
encounters: 1
lastSeen: 2026-08-13T00:00:00Z
---

# render.capture_open_level silently succeeds with a uniform black PNG

`render.capture_open_level` can return success, plausible output paths and correct dimensions for a populated lit level while writing a uniform black PNG. Six perspective captures of the active arena were byte-identical 17,709-byte images, and removing a temporary exposure volume did not change them. The response only proves that viewport pixels were read and encoded; it does not prove that the intended editor world was rendered or that the image contains usable scene information.

This is a silent false-success on a primary visual-review path. An agent trusts the success result, iterates against an unusable image, and lacks the viewport/world/render state needed to distinguish bad framing, a stale viewport, a world mismatch, and a failed scene redraw.

**Workaround:** decode every PNG externally and calculate pixel variance before trusting the capture. Separately query editor/viewport state, then retry manually. This is slow and still cannot prove which world the viewport rendered.

**Fix:** validate that the selected Level Editor viewport belongs to the active editor world before capture. Report the active world/package, viewport world, viewport type, view mode, game-view/realtime flags, applied camera state, and image luminance statistics. After readback, calculate mean luminance and variance; when the result is near-uniform black, perform at most one redraw retry and then return a typed `BLANK_CAPTURE` diagnostic unless the caller explicitly passes `allowBlank:true`. Restore every temporary viewport state on all exits. Add production-code regression coverage for blank classification and an editor integration test whose populated lit scene produces nonzero variance.

## History
- `#1-uniform-black-repro` `OPEN` reporter — Six perspective `render.capture_open_level` calls against a populated lit arena returned success and byte-identical uniform-black 17,709-byte PNGs. Removing a temporary exposure volume did not alter the output. Filed a high-severity silent false-success defect requiring world binding, state diagnostics, luminance/variance validation, one bounded retry, and `allowBlank` opt-out.
