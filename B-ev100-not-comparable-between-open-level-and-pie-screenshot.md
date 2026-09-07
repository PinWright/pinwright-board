---
id: B-ev100-not-comparable-between-open-level-and-pie-screenshot
title: "The same `ev100` produces a different exposure in `render.capture_open_level` and `editor.screenshot`, so the documented measure-then-pin recipe cannot be carried across the two verbs"
status: OPEN
severity: Medium
category: bug
tags: [render, capture, exposure, ev100, pie, screenshot, capture_open_level, docs]
encounters: 1
lastSeen: 2026-09-07T08:43:00Z
---

# One `ev100`, two verbs, two exposures — measured at an identical pose

`render.capture_open_level`'s wiki gives the recipe for making captures comparable: take one
`{mode:"auto"}` shot, read `viewport.exposure.ev100Equivalent`, pass that as `ev100` on the rest.
On the PIE path `{mode:"auto"}` returns no `ev100Equivalent` at all
(`B-editor-screenshot-pie-auto-exposure-returns-no-ev100`), so the only way to expose a PIE frame is
to measure on the level viewport and carry the number over to `editor.screenshot`. That carry does
not hold.

## Measurement (this checkout, 2026-09-07, `/Game/FPS/Test/T_AI`)

Identical camera pose, identical map, identical pinned EV100, both 640×360, one immediately after
the other. Pose was held fixed by placing the PIE view-target pawn at the same location/rotation the
level capture used:

    location {x:1501, y:-513, z:350}   rotation {pitch:-26, yaw:55, roll:0}   ev100 -4.13

| verb | mean luminance | blown fraction | response |
| --- | --- | --- | --- |
| `render.capture_open_level` | **0.598** | 0.007 | `pinned:true, fixed:true, ev100Equivalent:-4.13, adaptedSource:"fixedPin"` |
| `editor.screenshot` (PIE) | **0.743** | 0.094 | `pinned:true, viewCount:1, ev100:-4.13` |

Both report the pin applied. The gap is roughly two thirds of a stop, and the clipped fraction is
13× larger — far outside the ~1.3% viewport self-noise floor measured in
`B-exposure-pin-black-frame`. Means computed from the PNGs with a stdlib decoder; it agrees with the
engine's own `imageStats.meanLuminance` on the level capture to within 0.001 (0.598 vs 0.5998), so
the measurement route is not the variable.

Same-verb consistency is fine: within `capture_open_level`, `{mode:"auto"}` resolved to
`ev100Equivalent -4.13` with mean 0.5987, and re-shooting at `{mode:"fixed", ev100:-4.13}` gave
0.5998. It is only the crossing that breaks.

## What this does NOT establish

The game viewport is not the level viewport: `editor.screenshot` reports
`captureMode:"fixedSizeScenePlusUmg"`, the PIE world runs its own post-process chain (this map has a
`PostProcessVolume`, overrides off), and the two paths reported different `dpiScale`. So this is
**not** proof that the pin is misapplied on one side — the two viewports may legitimately resolve
different final exposure for the same EV100. The defect is at the workflow level: two verbs take a
parameter with the same name and the same documented meaning, both report `pinned:true`, and the
frames are not comparable. Nothing in either response or page says so.

## Ask

Whichever is true, say it in the response and on the page:

- If the two paths *should* match, they do not, and the game path needs fixing.
- If they legitimately cannot match — different post chain, UMG composite — then `editor.screenshot`
  should report the exposure the frame actually resolved to (an `ev100Equivalent` measured off the
  view, not an echo of the requested `ev100`), and `render.capture-exposure` should carry one line
  saying an EV measured on the level viewport is not transferable to the game viewport.

The echo is the part that misleads: `editor.screenshot` returns `ev100: -4.13` because that is what
was asked for, which reads as confirmation that the frame is at that exposure.

## Workaround

Bracket on the PIE path and pick by measurement instead of carrying a number over. On this map the
level viewport resolved −4.13 while the readable PIE value was near −3.0 to −3.2 (mean 0.436,
nothing blown, nothing crushed). Measuring the PNG is cheap: mean luminance, blown and crushed
fractions decode from the file with stdlib `zlib` alone, so a bracket costs captures rather than an
image read per candidate.

## Notes

- Sibling of `B-editor-screenshot-pie-auto-exposure-returns-no-ev100` (the PIE auto path returns
  `viewCount:0` and no `ev100Equivalent`). Together they close every route to a correctly exposed PIE
  frame: auto reports nothing, and a number measured elsewhere does not carry.
- `F-one-verb-for-exposed-and-aimed-pie-capture` is a third facet (aim vs expose). Note for whoever
  takes that one: the aim half is solvable today by moving the PIE player controller's view target
  and setting its control rotation, after which `editor.screenshot` captures an aimed frame with its
  exposure pin applied.
