---
id: B-showflag-cvar-override-contaminates-capture
title: "A `ShowFlag.*` cvar override forced ON by one agent renders a full-screen debug visualization into another agent's level capture — game view is true, the frame is two-thirds occluded, and nothing in the response says so"
status: DONE
severity: High
category: bug
tags: [render, capture_open_level, set_game_view, showflag, cvar, shared-state, concurrency, multi-agent, missing-warning, silent-contamination, review-hazard]
encounters: 2
lastSeen: 2026-08-28T09:10:00+05:00
---

# The capture reported `gameView: true` and returned a frame with `LIGHT FUNCTION ATLAS` debug chrome across the middle of it

Building the Atlantis avenue, the first acceptance capture came back with a **grey panel covering
roughly the central two-thirds of the frame**, captioned `LIGHT FUNCTION ATLAS`,
`Slot Resolution = 128x128 - Size = 4x4`, plus white body text down the right edge
(`Light Functions in atlas...`, `Local Lights sampling...`, `lights - /Game/Atl...`). The avenue
behind it was visible only as a thin border.

Every honesty field in the response was green:

```
gameView: true            splines: false           blank: false
hideEditorSprites: true   editorSprites.visible: false   editorSprites.restored: true
meanLuminance: 0.3979851390873726     litPixelFraction: 0.9993625217013888
toneLevelsUsed: 150       crushed: false           blownOut: false
warmup.settled: true
```

`meanLuminance 0.398` is a perfectly plausible number for this scene — the contaminated frame is
*brighter* than the clean one, so no luminance gate catches it either.

## Cause: a third suppression channel that nothing reports

The contamination is not an overlay show flag and not a billboard. It is an
**`EngineShowFlags` visualization forced on by a console variable**:

```
system.console.search {query: "LightFunctionAtlas"}
  -> ShowFlag.VisualizeLightFunctionAtlas   currentValue "1"   flags ["Cheat"]
```

Help text for that cvar family: *"Allows to override a specific showflag ... 0: force the showflag
to be OFF, 1: force the showflag to be ON, 2: do not override this showflag (default)"*. At `1` the
flag is forced ON for **every viewport in the process**, and game view does not clear it — the
visualization is a render pass, not an editor overlay.

Nobody on this build set it deliberately as a shared change: another agent was inspecting the
caustics light-function material (`/Game/Atlantis/Materials/M_LF_Caustics`) and left the override on.
It then contaminated captures for every other agent in the editor.

## Repro (measured pair, same pose, same pinned exposure, one cvar apart)

```js
// contaminated
system.console_command {command: "ShowFlag.VisualizeLightFunctionAtlas 1"}
editor.set_game_view {enabled: true}                       // -> gameViewEnabled: true, splines: false
render.capture_open_level {location:{x:-9000,y:0,z:260}, rotation:{pitch:-4,yaw:0,roll:0},
                           fov:60, width:768, height:768, hideEditorSprites:true,
                           exposure:{mode:"fixed", ev100:0}, filename:"ave_pose2_a"}

// clean — restore the engine default (2 = do not override)
system.console_command {command: "ShowFlag.VisualizeLightFunctionAtlas 2"}
render.capture_open_level { ...identical args..., filename:"ave_pose2_b"}
```

| frame | cvar | `meanLuminance` | `minLuminance` | `maxLuminance` | debug panel |
|---|---|---|---|---|---|
| `ave_pose2_a.png` | `1` | 0.3979851390873726 | **0.0** | 0.9999999999999999 | present |
| `ave_pose2_b.png` | `2` | 0.3273704023573317 | 0.0595717647058824 | 0.6154250980392156 | gone |

Both under `Saved/Screenshots/OpenLevel/`. `gameView: true` and `exposure.adaptedSource: "fixedPin"`,
`ev100 0` on both, so the cvar is the only variable. The `min 0.0 / max 1.0` pair on the
contaminated frame is the tell that survives into the numbers — pure black text and pure white text
are not in this scene's palette — but nothing in the response interprets it.

## Why the existing tickets do not cover this

- [`B-game-view-shared-state-no-capture-warning`](B-game-view-shared-state-no-capture-warning.md) is
  the case where a concurrent agent turns game view **off** and the capture measures
  `viewport.gameView: false` without warning. Here `gameView` was measured **`true`** and was
  genuinely true; the contamination arrived through a channel game view does not gate.
- [`B-set-game-view-does-not-suppress-spline-overlays`](B-set-game-view-does-not-suppress-spline-overlays.md)
  covers overlays disclosed through `notGovernedByGameView[]`. That array currently names exactly two
  escapes — `editorModeRender` and `debugDrawn` (verbatim from this session's response). A
  cvar-forced `ShowFlag.Visualize*` pass is a **third** escape and is named nowhere.

So this is the same family — "the frame is dirtier than the response admits" — with a new and
independently reachable mechanism.

## Suggested fix

The plugin already measures show-flag state for the ten overlay flags in
`Handlers/Editor/ViewportHandler.cpp` (readback added by `99eedf35`), so the machinery exists.

1. **Measure and report the `Visualize*` show flags that are ON at capture time.** `FEngineShowFlags`
   exposes every visualization flag by name; a capture whose viewport has any `Visualize*` /
   `VisualizeBuffer` / GPU-debug flag set should publish `visualizationFlags: ["VisualizeLightFunctionAtlas"]`
   and a `chromeWarning` naming it. That is the same measure-and-report house style as the existing
   `hideWarning` / `warmupWarning`, and it costs one loop over the flag set.
2. **Add the cvar-override channel to `notGovernedByGameView[]`** on `editor.set_game_view`, worded so
   the reader knows to check `ShowFlag.*` cvars: they are process-global, survive a game-view toggle,
   and in a shared editor are exactly the kind of state another agent leaves behind.
3. Optional and cheap: flag a capture whose `minLuminance == 0` **and** `maxLuminance == 1` while the
   scene is otherwise mid-tone. Pure-black-and-pure-white in the same frame is close to a signature
   for burned-in debug text.

Restoring the default is a one-liner (`ShowFlag.<Name> 2`), so the remedy is trivial **once you know
which flag** — the whole cost here is discovery. It took a `system.console.search` sweep over a
guessed keyword to find the name, after the picture made it obvious something was forced on.

## Impact

Any acceptance capture in a shared editor can be silently occluded by a debug visualization another
agent left enabled, and every field a reviewer is told to trust (`gameView`, `blank`,
`editorSprites`, `warmup.settled`, tone stats) reads clean. On this pass the first avenue acceptance
shot was two-thirds covered; had it been judged on the numbers alone it would have passed. It does
not take the editor down.

## History
- `#1-capture-surveys-forced-showflag-cvars` `IN-REVIEW` developer — Added the third channel as a measurement. `SurveyForcedShowFlagOverrides()` (`Handlers/Render/PreviewViewportCaptureUtils.cpp/.h`) walks `FEngineShowFlags::IterateAllFlags` — engine flags plus module-registered custom flags, so no hand-written table can drift — and reads each `ShowFlag.<Name>` console variable through `IConsoleManager`, collecting every one whose value is not the documented default of 2 together with its direction (0/1) and the priority that set it (`GetConsoleVariableSetByName`, the engine's own vocabulary rather than a local switch). It is read off the cvar and not off the viewport client because `EngineShowFlagOverride` ORs the force masks over the view's flags AFTER the client's own are copied (`Runtime/Engine/Private/ShowFlags.cpp`), so the client reports one thing and the renderer draws another; `FEngineShowFlags::IsForceFlagSet` was rejected as the detector because it cannot say in which direction, cannot name the priority, and its doc comment states the inverse of what its body returns. The survey runs once per capture, beside `bGameView`, and lands in `FViewportCaptureOutput` as `ForcedShowFlags` plus `bShowFlagOverridesMeasured`. `MakeViewportInfoObject` publishes it unconditionally as `viewport.showFlagOverrides {measured, forced[{name, cvar, value, direction, setBy}]}` with an `overrideWarning` present iff something is forced — so every capture verb reports it through the one shared builder. The warning names each cvar and the one-line remedy (`system.console_command "<cvar> 2"`), and states that the channel is process-global, survives a game-view toggle, and leaves `gameView`, `editorSprites` and the tone statistics reading clean. `editor.set_game_view` gained a third `notGovernedByGameView[]` entry (`showFlagCVarOverride`) naming the channel and pointing at `viewport.showFlagOverrides`. Reported, never refused: a shared editor condition the caller may knowingly accept must not fail the capture. The suggested `minLuminance == 0 && maxLuminance == 1` heuristic was deliberately NOT implemented — it is a correlation, and `docs/rpc-design.md` requires a warning to be derived from the thing it warns about; the cvar read is that thing. Test `PinWright.render.capture_contamination.ForcedShowFlagCVarIsSurveyedAndWarned` (`Tests/Render/TestCaptureContamination.cpp`) forces a `ShowFlag.Visualize*` cvar picked off the live registry (version-proof, and inert because those flags default off), asserts it is absent from the survey before, present with value/cvar/setBy while forced, and absent again after restore — then asserts the JSON: silent on a clean survey, `measured:false` when the survey never ran, and an `overrideWarning` naming the cvar for both the forced-on and forced-off directions. Written at the cvar's existing priority so the test cannot leave a ShowFlag.* pinned above where it found it.
- `#2-verified-fixed-behaviourally` `DONE` verifier — 2026-08-28. Ran the ticket's verbatim repro on the rebuilt binary at HEAD `b79ba53e` against the live editor, same pose and pinned exposure, one cvar apart. **Clean baseline** (cvar at its `2` default): `viewport.showFlagOverrides {measured:true, forced:[]}`, no warning — the survey is silent when there is nothing to report. **Forced**: `system.console_command "ShowFlag.VisualizeLightFunctionAtlas 1"`, then the identical capture → `viewport.showFlagOverrides.forced[0] = {name:"VisualizeLightFunctionAtlas", cvar:"ShowFlag.VisualizeLightFunctionAtlas", value:1, direction:"on", setBy:"Console"}` plus `overrideWarning` naming the cvar, stating the channel is process-global and survives a game-view toggle, that `gameView` / `editorSprites` / the tone statistics all read clean on the resulting frame, and giving the one-line remedy `system.console_command "<cvar> 2"`. **The contamination itself reproduced exactly and was vision-verified, not inferred:** `Saved/Screenshots/OpenLevel/verify_showflag_forced.png` carries the grey `LIGHT FUNCTION ATLAS` panel captioned `Slot Resolution = 128x128 - Size = 4x4` over roughly two-thirds of the frame, the scene surviving only as a border — while every honesty field the ticket lists still read green (`gameView:true`, `blank:false`, `editorSprites.visible:false`, `warmup.settled:true`), `meanLuminance` rose to 0.403523 from the clean frame's 0.351488, and the `minLuminance 0.0` / `maxLuminance 0.9999999999999999` signature the ticket predicted was present. The frame is still contaminated — the fix does not and should not suppress a process-global cvar another agent set — but the response now names the cause, which is what the ticket asked for. Fix item 2 confirmed live: `editor.set_game_view` returns a third `notGovernedByGameView[]` entry, `showFlagCVarOverride`, explaining the 0/1/2 semantics, that the override is ORed over the view's flags AFTER the viewport's own are copied, and pointing at `viewport.showFlagOverrides`. The suggested `min==0 && max==1` luminance heuristic was deliberately not implemented (developer entry `#1`); the cvar read supersedes it. **The forced-OFF direction was checked too, not assumed from the forced-ON case:** `ShowFlag.Bloom 0` then an identical capture returned `forced[0] = {name:"Bloom", cvar:"ShowFlag.Bloom", value:0, direction:"off", setBy:"Console"}` with the same `overrideWarning`, so the survey reads the cvar rather than looking for Visualize* names. Both cvars were restored to `2`, and the `VisualizeLightFunctionAtlas` restoration was confirmed by `system.console.search` before this entry was written.
