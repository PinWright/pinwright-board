---
id: B-no-way-to-capture-pie-pixels
title: "No verb can capture what a running PIE session renders: editor.screenshot returns an all-black game viewport, and after editor.eject the level viewport shows PIE but render.capture_open_level refuses it with VIEWPORT_WORLD_MISMATCH"
status: IN-REVIEW
severity: High
category: bug
tags: [render, capture_open_level, editor-screenshot, pie, eject, viewport, world-mismatch, visual-review]
encounters: 1
lastSeen: 2026-09-02T21:00:00Z
---

# A PIE session cannot be photographed

Visual review of anything that only exists while the game runs — AI behaviour, VFX in flight,
physics, a HUD under load — needs a picture of the PIE viewport. On this host there is no route to
one. Both available paths fail, in opposite directions.

## Path 1 — `editor.screenshot` during PIE: black frame, reported as success

```
editor.play {numClients:1, netMode:"standalone"}
editor.possess {actorName:"SpecEye"}          -> possessedPawnPath ...DefaultPawn_0, success
editor.screenshot {filename:"ai_pie_eye_01.png"}
-> {"width":1364,"height":979,"path":"...Saved/Screenshots/ai_pie_eye_01.png","captureSource":"gameViewport"}
```

The file is written and is essentially pure black: a faint blue line along the top edge and two
barely-visible dark shapes bottom-right, nothing else. Waiting 7 s of live PIE and re-capturing gave
an identical black frame. The session was genuinely running — a `python.execute` probe of the same
PIE world in the same seconds returned live actors with live transforms, and the AI had already
moved ~2000 uu from its spawn points.

Note `editor.screenshot`'s documented `BLANK_CAPTURE` guard did **not** fire: the page says "a
viewport that has never presented a frame reads back as an entirely empty surface. The job now fails
with `BLANK_CAPTURE` and writes no file". Here it wrote the file and reported success, so either the
frame is not *quite* uniform enough for the guard or the guard is not on this path. Either way the
caller gets a success and a black PNG.

## Path 2 — `editor.eject`, then capture the level viewport: refused

`editor.eject` works and does the right thing — the Level Editor viewport starts rendering the PIE
world:

```
editor.eject {} -> {"success":true,"ejected":true,"toggleQueued":true}
render.capture_open_level {filename:"...", location:..., rotation:...}
-> [VIEWPORT_WORLD_MISMATCH] Active Level Editor viewport does not render the active editor world.
   {"activeWorld":"/Game/FPS/Test/T_AI.T_AI",
    "viewportWorld":"/Game/FPS/Test/UEDPIE_0_T_AI.T_AI", "worldMatches":false, ...}
```

The guard is doing exactly what it was written to do — it exists to catch a *stale* viewport — but
here the mismatch is the intended state: the viewport is showing the simulate world on purpose,
which is the only surface that renders a running session. The check cannot tell "wrong world by
accident" from "PIE world by design".

`editor.screenshot` after eject does not help either: it still reports
`captureSource: "gameViewport"` and still returns black, and `editor.set_camera` (which targets the
Level Editor viewport) does not move what that capture shows.

## Expected

One of these, so that PIE is photographable at all:

- `render.capture_open_level` gains an opt-in — `allowPieWorld: true`, or an automatic pass when
  `viewportWorld` is a `UEDPIE_*` world of the same map — so the ejected/simulate viewport can be
  captured with the caller's pose and exact size. This is the smallest change and the most useful:
  it brings `width`/`height`/`location`/`rotation`/`exposure` to PIE captures.
- Or `editor.screenshot` grows a working game-viewport readback (and, failing that, actually raises
  `BLANK_CAPTURE` instead of writing a black PNG and reporting success).

## Impact

`visual-review.md` and this project's own `PLAN.md` make looking at a capture the acceptance gate
("a capture you have not looked at with your own vision does not count"). For any stream whose work
only exists at runtime, that gate is currently unreachable: the AI stream finished a Behavior Tree,
perception, cover EQS and a ragdoll death and could not produce a single frame of it running. The
fallback — probing the live world through `python.execute` and reading actor transforms and Blueprint
variables — proves *that* things happen but can never show what they look like.

**Workaround:** none for pixels. Runtime state can be read with
`unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem).get_game_world()` plus
`GameplayStatics.get_all_actors_of_class`, which is how this session verified anything at all.

severity rationale: impact=hard blocker with no workaround for the whole class of runtime-only work
(the documented acceptance gate cannot be satisfied) x reach=every stream that runs PIE -> High

## Fix

TRUE in the narrowed `#3` form. `editor.screenshot` had no exposure input and its game-viewport
readback inherited the possessed camera's final post-process blend; separately,
`render.capture_open_level` rejected every viewport/editor world mismatch without distinguishing an
intentional PIE world after eject. The screenshot handler now accepts the shared fixed-EV100
`exposure` contract through a draw-scoped, last-priority view extension. The extension sets the
view family's native fixed-EV100 override and matching finalized scene-view fields after camera and
other post-process blends, reports the measured application, and releases without changing
persistent camera state. The open-level handler now has a
default-off `allowPieWorld` exception that passes only for an `EWorldType::PIE` viewport whose map
matches the editor map after PIE-prefix removal, with typed mismatch reasons for every refusal.

Changed `Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp`,
`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp`,
`Source/PinWright/Private/Handlers/Render/OpenLevelCapture.h`,
`Source/PinWright/Private/Utils/ScreenshotUtils.h/.cpp`,
`Source/PinWright/Private/Handlers/ErrorCodes.h`, `Source/PinWright/Private/Tests/TestWorldUtils.h`,
editor/render wiki overlays, and tests. Tests:
`PinWright.editor.screenshot.Exposure.FixedEv100PostProcess`,
`PinWright.editor.screenshot.FixedSize.Contract`,
`PinWright.editor.screenshot.FixedSize.PieUmgComposite`, and
`PinWright.render.capture_open_level.AllowPieWorldGuardContract`.

Deliberately unchanged: the native-size `editor.screenshot` surface and its strict all-zero
`BLANK_CAPTURE` rule; history `#3` established that this readback works once camera exposure is
controlled. The PIE-world exception is explicit and same-map only rather than trusting any
`UEDPIE_` package name.

## History
- `#1-filed` `OPEN` reporter — Hit at the end of the FPS AI stream's only world-lock slot, on UE 5.8 /
  `EAContentExamples58`. Sequence and verbatim responses above. The black frame is reproducible: two
  captures 7 s apart, before and after `editor.possess`, all identical. The `python.execute` probe run
  between the two captures returned three live `BP_EnemyCharacter` actors at
  `(1871,1757,91)`, `(1805,1802,91)`, `(1794,1897,57)` — i.e. the world was simulating and the AI had
  moved roughly 2000 uu from its spawn points while the viewport was reading back black. After
  `editor.eject` the level viewport did switch to `UEDPIE_0_T_AI` (the error payload proves it), so
  the pixels exist and are being drawn somewhere; only the capture verbs cannot reach them. No plugin
  source read.
- `#2-route-found-screenshot_window-plus-exposure` `OPEN` reporter — **There is a working route, and
  the black frames are an exposure fault rather than a broken capture path.** Found while reviewing
  the same stream's AI as a critic, same host, same map (`/Game/FPS/Test/T_AI`), UE 5.8. Sequence:
  `editor.play {numClients:1, netMode:"standalone"}` -> `editor.eject` -> `editor.set_camera {...}`
  -> **`editor.screenshot_window {window_title:"Unreal Editor"}`**. That writes a 1887x1209 PNG of
  the whole editor window whose Viewport 1 pane is the *ejected PIE world*, correctly lit and at the
  pose `editor.set_camera` was given — the Outliner in the same frame reads `T_AI (Play In Editor)`
  and the PIE pause/step/stop/eject buttons are in the toolbar, so the frame is provably from the
  running session. Three files: `Saved/Screenshots/EditorWindow/aicritic_pie_eye_fight.png` (1.9 MB,
  two AI characters at 190 cm eye height mid-fight), `aicritic_pie_standing_open.png` (1.9 MB), and
  the useless `aicritic_pie_eject_eye.png` (265 KB, black) taken *before* the exposure fix.
  **The exposure fix is one console command:** `editor.console_command {command:"ShowFlag.EyeAdaptation 0"}`
  before the capture, and `... 1` afterwards to restore the shared editor. Without it the level
  viewport's auto-exposure sits mis-adapted (measured on this map through
  `render.capture_open_level {exposure:{mode:"auto"}}`: `adapted 0.00208`, `ev100Equivalent 8.9`,
  `meanLuminance 0.00033` — i.e. black; the same camera at `{mode:"fixed", ev100:-1.8}` gave
  `meanLuminance 0.26`). `r.EyeAdaptation.MethodOverride 3` did **not** fix it; `ShowFlag.EyeAdaptation 0`
  did. That also re-reads `#1`'s Path 1: the `editor.screenshot` PIE frame is not blank-because-nothing-
  presented — my copy (`Saved/Screenshots/aicritic_pie_01.png`) contains the game's *debug line*
  primitives (red trace lines and a green hit segment, drawn by the AI weapon's
  `LineTraceSingle {DrawDebugType: ForDuration}`) on an otherwise black field, so the readback reached
  the renderer and the scene colour itself tone-mapped to zero. Worth re-testing `editor.screenshot`
  with `ShowFlag.EyeAdaptation 0` before treating that path as broken.
  **What is still missing, and why this stays OPEN:** the working route is a *whole-window* capture,
  so every frame carries editor chrome (menu bar, toolbars, Outliner, Details, status bar) around a
  ~1364x970 viewport pane, at whatever size the window happens to be. There is no `width`/`height`,
  no `exposure`, no `hideEditorSprites`, no `subject`/`framing`, no `imageStats`/`blank` guard, and no
  way to get a clean frame for a side-by-side against a reference still without cropping outside the
  tool. The `#1` proposal stands and is the right fix — `render.capture_open_level` with
  `allowPieWorld: true`, or an automatic pass when `viewportWorld` is a `UEDPIE_*` world of the same
  map — but the severity is now "no *clean, sized* PIE capture" rather than "no PIE pixels at all",
  and the `ShowFlag.EyeAdaptation` note belongs in `visual-review.md` either way, since a
  mis-adapted viewport silently blackens `render.capture_open_level` too. No plugin source read.
- `#3-editor-screenshot-is-clean-once-the-pawn-camera-is-biased` `OPEN` reporter — **The clean, sized
  PIE route exists after all: `editor.screenshot` is not broken, the pawn's camera is.** FPS
  PLAYER-critic slot, 2026-09-02T21:49-21:57Z, `/Game/FPS/Test/T_Player`. `editor.screenshot` during
  PIE returned `captureSource:"gameViewport"`, 1364x979, **no editor chrome** — the exact thing `#2`
  says is missing — but the first frame was black
  (`Docs/fps/evidence/player/critic-00_idle_methodoverride_minus1.png`). Two things then falsified the
  cvar theory: `r.EyeAdaptation.MethodOverride` was already globally pinned at **3 (Manual)** when I
  arrived (not `-1`), and setting it to `-1` changed nothing — the frame stayed black. What fixed it
  was overriding exposure **on the possessed pawn's own CameraComponent**, which owns the final
  post-process blend and therefore outranks both the level PPV and the metering cvar:

      property.set {objectPath: "<PIE world>:PersistentLevel.<Pawn>.Camera",
                    propertyName: "PostProcessSettings.bOverride_AutoExposureBias", value: true}
      property.set {objectPath: "...Camera",
                    propertyName: "PostProcessSettings.AutoExposureBias", value: 11}

  The very next `editor.screenshot` was fully exposed, full tonal range, chrome-free
  (`critic-01_idle_bias11.png`) — and it survives `editor.stop` / `editor.play` only if re-applied,
  because the PIE world is re-duplicated each run. This is runtime-only: `markedDirty:false`, nothing
  saved, no world package dirtied (`editor.list_dirty_packages` clean before release).
  **What this changes for the ticket:** the missing capability is narrower than `#2` states — it is
  not "no clean PIE capture" but "no `exposure` parameter on `editor.screenshot`". Every caller must
  currently know to reach into the possessed pawn's camera. The fix is to give `editor.screenshot`
  the same `exposure: {mode, ev100}` block `render.capture_open_level` already has, applied to the
  game viewport's view family and restored afterwards; failing that, `visual-review.md` should carry
  the `AutoExposureBias` recipe beside the `ShowFlag.EyeAdaptation` one, since `#2`'s show-flag route
  costs the whole window and this one does not. No plugin source read.
- `#4-editor-screenshot-exposure-and-pie-opt-in` `IN-REVIEW` developer — Added shared fixed-EV100
  exposure parsing, scoped local-camera application/restoration and evidence to `editor.screenshot`;
  added a default-off `allowPieWorld` gate to `render.capture_open_level` that accepts only a real PIE
  viewport world while preserving the stale-world refusal for every other mismatch.
- `#5-final-view-exposure-and-same-map-guard` `IN-REVIEW` developer — Moved the game-viewport EV100
  pin to finalized scene views so later post-process blends cannot override it; extracted and
  behaviorally tested the world guard, including same-map PIE acceptance and another-map refusal.
- `#6-suite-3-handler-registry` `IN-REVIEW` developer — Suite-3 follow-up: `ViewportHandler` now
  fully adopts `ErrorCodes` constants after `editor.screenshot` introduced registry usage; the wiki
  handler test now uses native paired `width`/`height` plus `sceneOnlyFallback` disclosure instead
  of the obsolete fixed-size redirect/native-resolution fallback contract.
