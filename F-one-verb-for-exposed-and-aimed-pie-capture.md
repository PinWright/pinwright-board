---
id: F-one-verb-for-exposed-and-aimed-pie-capture
title: "No single verb can take a correctly exposed PIE frame containing a named actor — screenshot_window aims but cannot expose, editor.screenshot exposes but cannot aim"
status: OPEN
severity: Medium
category: feature
tags: [capture, screenshot, exposure, pie, viewport]
---

# No single verb can take a correctly exposed PIE frame containing a named actor

Two capture verbs each solve half of the same job, and the halves do not compose.

| | aims at a chosen pose | exposure control | editor chrome |
| --- | --- | --- | --- |
| `editor.screenshot_window` | **yes** — follows `editor.set_camera` | **no** parameter | yes, whole window |
| `editor.screenshot` | **no** — `captureSource: gameViewport`, ignores `set_camera` | **yes** — `exposure {mode:"fixed", ev100:N}`, verified `pinned:true` | no, clean frame |

So "a correctly exposed frame containing this actor" is currently not expressible in one call.

## How this actually plays out

Photographing an AI firing a rifle, over four consecutive builds of this project:

- Through `screenshot_window`: the actor is in frame, and the muzzle flash — a strong local light —
  blows the shot out, because the only exposure lever available is `ShowFlag.EyeAdaptation 0`, which
  pins a fixed EV with no way to choose it. Leaving auto-exposure on instead makes successive frames
  incomparable, which the `editor.screenshot` docs themselves warn about.
- Through `editor.screenshot {exposure:{mode:"fixed", ev100:1}}`: exposure is correct and repeatable
  and the frame is clean, and the actor is not in it — twice, even with the world frozen at
  `set_global_time_dilation(0.0001)` so the subject could not move between framing and shutter, and
  with the camera pose computed from the pawn's own transform.

The workaround is to accept one defect per frame: a blown-out frame that contains the subject, or a
well-exposed frame of empty scenery. Neither is usable evidence.

## Ask

Either of these closes it; the first is smaller:

1. **`exposure` on `editor.screenshot_window`** — the same `{mode:"fixed", ev100:N} | {mode:"auto"}`
   spelling already implemented for `editor.screenshot`, applied to the level-editor viewport for the
   duration of the grab and restored after. The rendering path already supports a scoped exposure
   override; this is plumbing an existing parameter to a second verb.
2. **Aiming for `editor.screenshot`** — a `location`/`rotation` pair, or a `viewTarget` actor name,
   applied to the game viewport for the shot. `editor.console_command` already accepts a
   `world:'game'` selector, so a `world` selector on `editor.set_camera` would achieve the same and
   be useful on its own.

A `cropToViewport` option on `screenshot_window` would be a welcome extra — the editor chrome is
dead weight in every frame — but it is not what blocks the job.

## Notes

- Adjacent to `B-no-way-to-capture-pie-pixels` `#7` (commit `3dce460`), which records the same split
  as the residue of that bug. Filed separately as a feature because the bug is "PIE pixels are
  uncapturable" (largely fixed — both routes now return real frames) whereas this is "the two routes
  have disjoint capabilities", which needs a design decision rather than a fix.
- Also adjacent to `F-pie-capture-fixed-size-with-umg` and `F-editor-viewport-screenshot`, neither of
  which mentions exposure.
- Workaround available to a project, and what this stream is doing next: put a `PostProcessVolume`
  in the level with manual exposure so the scene's own EV is sane, then use `screenshot_window`. That
  fixes it per-level by changing the content, which is not a substitute for a capture-time control.
