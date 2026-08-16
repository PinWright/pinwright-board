---
id: B-set-game-view-does-not-suppress-spline-overlays
title: "set_game_view reports a clean frame but does not suppress spline overlays — a water spline's white dashed bank lines render into captures and have twice been on the verge of being reported as map content"
status: OPEN
severity: Medium
category: bug
tags: [editor, viewport, set_game_view, capture, overlays, splines, water, false-feature, review-hazard]
---

# The frame is warned about, not cleaned

`editor.set_game_view` returned `gameViewEnabled: true` and the capture reported `gameView: true`,
and a water spline still rendered **a white dashed line down each bank** into the frame.

That artifact is an editor overlay, not map content — and it has now twice come close to being
written up as a placement feature. Recorded in the host project at
`Docs/map/reference_tile_compare.md:236-239`:

> **`set_game_view` does not remove the water spline.** `gameViewEnabled: true` and the capture
> reporting `gameView: true`, and the river still renders with a white dashed line down each bank in
> `tc_probe_centre.png`. This is the exact artifact a previous reviewer nearly logged as foam. It is
> an editor overlay, not map content — **do not report it as a placement feature**.

"Nearly logged as foam" is the cost: a review pass reading a rendered frame has no way to tell an
editor overlay from authored geometry, and the verb's success response actively suggests there is
nothing left to tell apart.

## What is already fixed, and what is not

`99eedf35` ("Report the overlay flags game view actually governs") closed the **honesty** half:
`Handlers/Editor/ViewportHandler.cpp` now reads back ten overlay show flags off `EngineShowFlags` and
emits `notGovernedByGameView[]` **unconditionally** (`:706`), so the response can no longer be read
as a blanket clean-frame claim. Test: `Tests/EditorOps/TestSetGameViewOverlayReporting.cpp`
(157 lines), asserting the array is present at `:137-138`. On `origin/master`.

**The suppression half is open**, and deliberately so. The commit's load-bearing sentence:

> The verb does not force-clear the flag; that would be an unannounced global mutation it cannot
> restore.

That reasoning is sound — silently clearing a global show flag the verb cannot put back is worse than
disclosing it — but it leaves the actual hazard in place: **a spline can still render into a capture.
It is warned, not suppressed.** A caller who does not read `notGovernedByGameView[]`, or an
image-diffing pass that never sees the JSON at all, is exactly as exposed as before.

## Fix shape

The disclosure is the floor, not the ceiling. Options, in rough order of cost:

1. An **opt-in** parameter (e.g. `suppressOverlays: true`) that clears the named flags for the
   duration of a capture and restores them afterwards — the restore obligation is what
   `99eedf35` correctly refused to take on implicitly, but it is takeable explicitly, scoped to one
   call.
2. Have the **capture** verbs own it rather than `set_game_view`: a capture is a bounded operation
   with a natural restore point, which `set_game_view` (a persistent mode toggle) is not.
3. If neither is wanted, say so on the wiki page in the caller's language — "splines and other
   non-game-view-governed overlays WILL appear in your frame; here is how to identify them" — so a
   reviewer reading a PNG is warned in the place they actually look.

## Related

- `B-set-game-view-keeps-editor-billboards` — the same class, different overlay.
- `B-set-view-mode-writes-only-active-projection-slot` — the sibling capture-fidelity defect found in
  the same pass.

## History
- `#1-warned-not-suppressed` `OPEN` reporter — `editor.set_game_view` reported `gameViewEnabled: true` and the capture reported `gameView: true`, yet a water spline still rendered a white dashed line down each bank into the frame (`tc_probe_centre.png`). Recorded in the host project at `Docs/map/reference_tile_compare.md:236-239`, which notes this is **"the exact artifact a previous reviewer nearly logged as foam"** — i.e. an editor overlay twice on the verge of being reported as authored map content, because a review pass reading a rendered frame cannot distinguish overlay from geometry and the verb's success response implied there was nothing left to distinguish. **Half of this is already fixed and the split matters.** `99eedf35` (on `origin/master`) closed the honesty half: `Handlers/Editor/ViewportHandler.cpp` reads back ten overlay show flags off `EngineShowFlags` and emits `notGovernedByGameView[]` unconditionally (`:706`), so the response can no longer read as a blanket clean-frame claim, with `Tests/EditorOps/TestSetGameViewOverlayReporting.cpp` (157 lines) asserting the array's presence at `:137-138`. The suppression half is what remains, and the commit states why it was left: *"The verb does not force-clear the flag; that would be an unannounced global mutation it cannot restore."* That reasoning is right — silently clearing a global show flag with no restore is worse than disclosing it — but the hazard is unchanged for anyone who does not read the JSON, including any image-diffing pass that never sees it. Fix shape, cheapest first: an opt-in `suppressOverlays`-style parameter that clears the named flags for one call and restores them (the restore obligation is takeable explicitly even though it was rightly refused implicitly); or move the responsibility to the capture verbs, which are bounded operations with a natural restore point where `set_game_view` is a persistent mode toggle; or, if neither, say it plainly on the wiki page in the caller's language so a reviewer reading a PNG is warned where they actually look.
