---
id: B-capture-open-level-during-pie-silently-captures-editor-world
title: "`render.capture_open_level` with `allowPieWorld:true` during an active PIE session captures the EDITOR world and reports every quality signal green — the frame contains none of the live actors the caller aimed at"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_open_level, pie, allowPieWorld, silent-success, wrong-world, warning]
encounters: 1
lastSeen: 2026-09-08T15:51:00Z
---

# A capture aimed at a PIE actor returns a frame of the editor world, and says nothing

With PIE **active and paused**, `render.capture_open_level` accepted `allowPieWorld: true` and a
camera pose computed from a live PIE character, and returned a normal-looking frame of the *editor*
world at that pose — empty ground, because the live actor exists only in the PIE world.

    render.capture_open_level {filename:"b10_burst_paused.png", width:1280, height:720,
                               allowPieWorld:true,
                               location:{x:1626.4, y:691, z:241.1},
                               rotation:{pitch:-7.6, yaw:23, roll:0},
                               exposure:{mode:"fixed", ev100:-3}, hideEditorSprites:true}

Response, abridged — every signal a caller would check is green:

    "blank": false,  "crushed": false,  "blownOut": false,
    "imageStats": { "meanLuminance": 0.5875, "toneLevelsUsed": 200, "litPixelFraction": 1 },
    "viewport": { "warmup": { "settled": true, "settleRounds": 1 },
                  "exposure": { "pinned": true, "pinnedFrameUsable": true },
                  "aim": { "applied": true, "rotationErrorDegrees": 0, "locationErrorCm": 0 } },
    "worldMatches": true,
    "allowPieWorld": true,
    "capturedPieWorld": false,          <-- the only field that says what happened
    "pieWorldSameMap": true

The frame is real, correctly exposed, correctly aimed, and of the wrong world. Verified by reading
the PNG: no characters anywhere in it, while three were alive and fighting in PIE.

## Why this is a trap rather than a doc-reading failure

The page says `allowPieWorld:true` is for "capturing a Level Editor viewport that is still bound to a
PIE world **after eject**". Without an eject the viewport is still bound to the editor world, so
capturing that is arguably correct behaviour. The defect is that nothing says so:

- `allowPieWorld` reads as a *request* ("capture the PIE world") and is in fact a *permission*
  ("don't refuse if the viewport happens to be PIE-bound"). Passing it when the viewport is not
  PIE-bound is a silent no-op, not an error.
- `worldMatches: true` is the reassuring field, and it is true — of the editor world.
- `capturedPieWorld: false` is the one field carrying the bad news, is easy to miss among ~40
  response fields, and has no accompanying warning string, unlike almost every other condition this
  verb reports (`gameViewWarning`, `pinWarning`, `rangeWarning`, `viewDistanceWarning`,
  `cullingOriginWarning`, `hideWarning`, `restoreWarning`).

## Ask

When PIE is active and the capture is of the editor world, emit a warning field saying so and naming
the remedy — something like:

    "pieWorldWarning": "A PIE session is active but this capture is of the EDITOR world; the live
     PIE actors are not in this frame. Eject first (the level viewport then binds to the PIE world),
     or use editor.screenshot for game-viewport pixels."

That is the same shape as the `gameViewWarning` this verb already emits, and it turns a frame that
silently answers the wrong question into one line the caller cannot miss.

## Workaround

For live PIE pixels use `editor.screenshot`, aiming by moving the PIE player controller's view
target and setting its control rotation. To use `capture_open_level` on a PIE world, eject first so
the level viewport binds to it, then check `capturedPieWorld: true` on the response rather than
assuming the flag took effect.

## Notes

- Found while trying to follow the safer capture path suggested by
  `B-editor-screenshot-pie-gpu-pagefault-during-shader-late-association` — `capture_open_level` has
  the settle/warmup step `editor.screenshot` lacks, so it is the natural verb to reach for after
  that crash. It just cannot see the PIE world without an eject, and does not say it.
