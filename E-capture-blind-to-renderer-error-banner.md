---
id: E-capture-blind-to-renderer-error-banner
title: "render.capture_open_level publishes four channels of frame-contamination warnings but not the one that was actually burned into my frames: the renderer's own on-screen error text ('Video memory has been exhausted'), which no show flag, game view or hideEditorSprites removes"
status: IN-REVIEW
severity: High
category: enhancement
tags: [render, capture_open_level, capture, contamination, warning, acceptance-shot, vram, on-screen-message]
encounters: 2
costly: 2
lastSeen: 2026-09-08T15:06:00Z
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

- `#2-returned-by-second-stream` `OPEN` ENV reporter — Hit again 2026-09-08 15:05Z in an ENV
  build-08 world-lock slot on `/Game/FPS/Maps/FPS_Compound`, three days and one editor restart
  after `#1`, so the restart does not retire it. `render.capture_open_level` at the pinned
  `env_poses.py` pose 01 returned `blank:false`, `crushed:false`, `blownOut:false`,
  `toneLevelsUsed:256`, `litPixelFraction:0.99966`, `showFlagOverrides.forced:[]`,
  `editorSprites.visible:false`, `exposure.pinned:true` — every contamination channel clean — and
  the PNG carries `Video memory has been exhausted (377.549 MB over budget). Expect extremely poor
  performance.` in red across the upper third. Note the overrun is 377 MB here against 1197 MB in
  `#1`: the banner fires well below the earlier figure, so a caller cannot infer it from uptime or
  from a remembered threshold either. Cost: two acceptance frames discarded and the whole exterior
  re-shoot deferred to a fresh editor, spending the slot on placement only (`costly` 1 -> 2).
  **Severity Medium -> High by reach**, naming ENV and PLAYER-critic: the orchestrator reports the
  PLAYER critic hit the same banner in the slot immediately before mine, which is a second stream
  on the same verb — recorded as the orchestrator's report rather than my own measurement, since I
  did not see that stream's frames. Cost alone would not have bumped it (`costly` 2, below the
  threshold of 3). `#1`'s ask is unchanged and is still the fix: a `viewport.onScreenMessages`
  array from `GEngine->GetOnScreenDebugMessages()` plus a warning when it is non-empty. A second
  observation worth recording for whoever implements it: the banner is drawn into scene colour
  before readback, so it also corrupts `imageStats` — `meanLuminance` 0.2908 on this frame includes
  the red text, which is why no luminance-based heuristic should be used in place of reading the
  message list.

- `#3-on-screen-message-survey` `IN-REVIEW` developer — Added `Source/PinWright/Private/Utils/OnScreenMessageSurvey.h/.cpp` (`PinWrightOnScreenMessages`) and published `viewport.onScreenMessages[]` (`{severity, source, text, color}` rows) plus `onScreenMessageCount`, `screenMessagesEnabled`, `mapWarningsSuppressed` and an `onScreenMessageWarning` from `render.capture_open_level` (`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp`) and `editor.screenshot`, both branches (`Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp`). Docs: `Docs/wiki-src/render.md`, `Docs/wiki-src/editor.md`. **Two corrections to this ticket's stated mechanism, both verified against `C:\UE_5.8\Engine\Source`.** (1) `GEngine->GetOnScreenDebugMessages()` **is not an API in UE 5.8** — `ScreenMessages` and `PriorityScreenMessages` are private `UEngine` members with no accessor and no `UPROPERTY` (`Engine.h:2186-2206`), so they are unreadable by call and by reflection; anything added via `GEngine->AddOnScreenDebugMessage` is therefore NOT covered and this fix does not claim it. (2) The VRAM banner is **not on that path at all** — `SceneRendering.cpp:4802-4806` prints it inline from `GDemotedLocalMemorySize` under the `r.DemotedLocalMemoryWarning` cvar, so a delegate-only collector would have closed nothing. What is covered, and is what the reported frames carried: `FCoreDelegates::OnGetOnScreenMessages` (public, game-thread-broadcast, severity-tagged) and the demoted-local-memory banner, whose condition and format string are reproduced verbatim from the two public globals. Not covered and stated as such in the code and the wiki: the renderer's private `FSceneRenderer::OnGetOnScreenMessages` (`Renderer/Private/SceneRendering.h`, render-thread only), which is where `#1`'s second yellow line about competing directional lights comes from. The engine's own two suppression gates (`GAreScreenMessagesEnabled`, `GEngine->bSuppressMapWarnings`) are honoured and published, so an empty array is a measurement rather than a false clean; an absent key means no survey ran. Tests `PinWright.render.capture_on_screen_messages.EngineMessagesArePublishedAndFlagged` and `.VramBannerMatchesTheRendererText` (`Source/PinWright/Private/Tests/Render/TestCaptureOnScreenMessages.cpp`). Counterfactual: with the collector reverted — i.e. an unmeasured survey, which is what every capture response carried before this — `AddOnScreenMessageFields` adds no `onScreenMessages` key and no `onScreenMessageWarning`, which the first test asserts directly; and dropping the demoted-local-memory branch from `Survey()` makes `VramBannerMatchesTheRendererText` fail on the exact-string comparison against the renderer's format output, restoring the state in which ten acceptance frames read clean on every field.
