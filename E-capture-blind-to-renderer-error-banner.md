---
id: E-capture-blind-to-renderer-error-banner
title: "render.capture_open_level publishes four channels of frame-contamination warnings but not the one that was actually burned into my frames: the renderer's own on-screen error text ('Video memory has been exhausted'), which no show flag, game view or hideEditorSprites removes"
status: OPEN
severity: Medium
category: enhancement
tags: [render, capture_open_level, capture, contamination, warning, acceptance-shot, vram, on-screen-message]
encounters: 1
costly: 1
lastSeen: 2026-09-05T19:00:00Z
---

# The capture verb warns about four kinds of contamination and misses a fifth

`render.capture_open_level` goes to real trouble to tell a caller when the pixels are not a
picture of the level. It publishes `viewport.showFlagOverrides.forced` with an `overrideWarning`,
`overlayShowFlags` with an `overlayWarning`, a `gameViewWarning`, `editorSprites`, plus
`blank` / `crushed` / `blownOut` and a full `imageStats`. That is the right instinct, and on this
project it has caught a forced `ShowFlag.EyeAdaptation` several times.

It has no channel for the renderer's **own on-screen error text**, which is drawn into the scene
colour before readback.

## Measured

UE 5.8, shared editor after long uptime with six agent streams. Every frame from a 15-minute
acceptance slot came back with this burned across the upper third, in red:

```
Video memory has been exhausted (1197.488 MB over budget). Expect extremely poor performance.
```

and, while a second directional light existed, a second line in yellow:

```
Multiple directional lights are competing to be the single one used for forward shading,
translucent, water or volumetric fog. ...
```

Every published field read clean on those same frames:

```
showFlagOverrides.forced: []      overlayShowFlags: ... game: true
editorSprites: { hideRequested: true, visible: false, restored: true }
blank: false   crushed: false   blownOut: false   litPixelFraction: 0.9998
imageStats.meanLuminance: 0.307   toneLevelsUsed: 177
```

Nothing removes it from the caller's side: it is not a show flag, `hideEditorSprites` governs
billboards only, and `editor.set_game_view` does not touch it. `collect_garbage`,
`r.Streaming.FlushTextureStreaming` and a pool resize did not clear the underlying condition.

## What it cost

Ten acceptance captures were taken, read, and then **discarded** — a red error banner across the
frame disqualifies a blind A/B against reference stills, so the whole set had to be re-shot in a
fresh editor, at the cost of a world-lock slot and a queue wait. Had the response said so, the
slot would have been spent on the restart instead of on the captures.

The wider point: an agent that trusts the response cannot know. The verb's own contamination
reporting is what makes it trustworthy for acceptance work, and this is a gap in exactly that
contract, not a new category of problem.

## What would close it

The engine already holds these strings — `GEngine->GetOnScreenDebugMessages()` for the timed and
keyed messages, and the VRAM-over-budget line is emitted through the same on-screen path. A
`viewport.onScreenMessages` array (count plus the text, as `showFlagOverrides` does for cvars),
with a warning when it is non-empty, would be consistent with everything else the verb reports.
A caller could then decide to re-shoot rather than discover it by eye.

Detecting it from pixels is not required and would be the wrong mechanism — the strings are
available directly.

## History
- `#1-filed` `OPEN` ENV reporter — Observed 2026-09-05 building the FPS compound map (map as forcing function; host `CLAUDE.md` § "What this project is for"), UE 5.8, PinWright at this checkout's HEAD after the wave-4-9 rebuild. Ten `render.capture_open_level` acceptance frames came back with the renderer's "Video memory has been exhausted (1197.488 MB over budget)" burned into them in red, while `showFlagOverrides.forced` was `[]`, `editorSprites.visible` was `false`, `gameView` was `true`, and `blank`/`crushed`/`blownOut` were all clean — every contamination channel the verb publishes read fine. `hideEditorSprites` governs billboards only and game view does not cover on-screen messages, so there is no caller-side way to suppress or even detect it short of opening the PNG. The set was discarded and re-shot in a restarted editor, costing a world-lock slot. Asked for a `viewport.onScreenMessages` array sourced from `GEngine->GetOnScreenDebugMessages()` plus a warning when non-empty, matching how `showFlagOverrides` already reports forced cvars.
