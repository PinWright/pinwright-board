---
id: B-game-view-shared-state-no-capture-warning
title: "Game view is shared per-viewport state a concurrent agent can turn off under you, and the capture verbs measure `viewport.gameView: false` without warning — editor chrome lands in an acceptance capture that reports success"
status: OPEN
severity: High
category: bug
tags: [render, capture_open_level, game-view, shared-state, concurrency, multi-agent, missing-warning, silent-contamination]
encounters: 1
lastSeen: 2026-08-27T18:47:15+05:00
---

# A capture taken after a confirmed `editor.set_game_view {enabled: true}` came back with editor chrome in it, `viewport.gameView: false`, and no warning

Game view is per-**viewport** state, not per-call. In an editor shared by several
agents it is a global that any agent can move under any other, at any moment
between one call and the next. So a caller can do everything right — assert game
view, confirm `gameViewEnabled: true`, then capture — and still get a frame with
editor chrome in it, because a concurrent agent turned it off in between.

The capture response **measures this and says nothing about it.** It publishes
`viewport.gameView: false` and returns success. There is no warning field, no
error, and no way to fail the capture instead of returning a contaminated frame.
`editorSprites` reports a clean `{hideRequested: true, visible: false, restored:
true}` — because `hideEditorSprites` did its job on billboards and the world-axis
gizmo is not a billboard.

This is the first defect on this board about **shared editor state between
concurrent agents**, as distinct from state a single caller forgot to set.

## Root cause (guilty source line)

The measurement is already taken.
`Plugins/PinWright/Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:1585`:

```cpp
        Viewport->SetBoolField(TEXT("gameView"), ViewportClient.IsInGameView());
```

(The preview-capture path publishes the same field at
`Handlers/Render/PreviewViewportCaptureUtils.cpp:2833`.)

**Only the warning is missing**, and the house pattern for exactly this already
exists twice in the same code:

- `Handlers/Render/PreviewViewportCaptureUtils.cpp:2512` emits `hideWarning` on
  the sprite-hiding block.
- `Handlers/Editor/ViewportHandler.cpp:706` emits `overlayWarning` from
  `editor.set_game_view` when game view is on and `Splines` is still set.

A grep of the whole plugin returns **zero** occurrences of `gameViewWarning` and
zero of `requireGameView`. So the capture verbs measure the one fact that
invalidates the frame and decline to raise it, while the neighbouring verb raises
a warning about a strictly less severe condition.

## Verbatim repro

With several agents sharing one editor:

```js
call("editor.set_game_view", { enabled: true })     // -> gameViewEnabled: true
// ... another agent calls editor.set_game_view {enabled:false}, or anything that toggles it ...
call("render.capture_open_level", { hideEditorSprites: true, /* pose, pinned exposure */ })
//  -> success, viewport.gameView: false, editor chrome in the pixels
```

Observed live on `/Game/Maps/Atlantis`, 2026-08-27, UE 5.8, this checkout, while
two placement agents were seating actors.

## Evidence

`Saved/Screenshots/OpenLevel/review_p4_temple_oblique.png`, first take: the
world-axis gizmo visible at bottom-left, `viewport.gameView: false`,
`meanLuminance` 0.331572. Re-asserting game view and re-shooting the identical
pose gives 0.343396 and no gizmo. (The file was overwritten by the retake, so the
surviving artefact is the response pair, not the image.)

**The measurement that shows a third party did it, not the caller.** Between one
capture and the next, `editor.set_game_view {enabled: true}` reported
`previous.gameViewEnabled: false` with `previous.overlayShowFlags` showing
`splines`, `grid`, `volumes`, `selection`, `selectionOutline` and `lightRadius`
**all back on** — the full editor set. The reporting agent had not turned any of
them off.

## Impact

Quiet and corrosive rather than fatal:

- Editor chrome lands in a capture presented as an **acceptance shot**.
- Because game view also gates component visualizers, a spline or landscape
  overlay can appear as a line running through the level that a reviewer reads as
  content. `render.capture_open_level`'s own wiki names exactly this failure for
  the `splines` flag — *"a line running along a river in a capture is that
  overlay, not foam"* — but frames it as something the caller forgot, not
  something a concurrent caller can undo **after the caller got it right**.
- It silently breaks pass-to-pass comparability, which is the entire premise of a
  fixed review pose set: two frames of an unchanged scene differ by the chrome
  alone. The 0.331572 → 0.343396 shift above is precisely that.

## What it should do

1. **`gameViewWarning` on every capture verb** — `render.capture_open_level`, the
   `camera.*` verbs, and `render.capture_annotated` — emitted when
   `viewport.gameView` is `false`, in exact parallel with the existing
   `hideWarning` (`PreviewViewportCaptureUtils.cpp:2512`) and `overlayWarning`
   (`ViewportHandler.cpp:706`). The measurement is already taken at
   `RenderHandler.cpp:1585`; only the warning is missing, and a warning is what
   makes a caller look.
2. **A `requireGameView: true` parameter** that fails the capture rather than
   returning a frame with chrome in it. For an acceptance shot, no image is more
   useful than a contaminated one that looks fine.

## The general question this opens

This is the first defect filed here about a **shared global between concurrent
agents**, and the same question should be asked of every other global the capture
path depends on: view mode, `r.ViewDistanceScale`, exposure, and the scalability
cvars behind `B-setup-volumetric-fog-enabled-true-while-cvar-off`. Each is state
one agent sets and another can move, and in each case the capture verb either
measures it already or could.

## Workaround

Call `editor.set_game_view {enabled: true}` immediately before **every** capture,
not once per burst, and read `viewport.gameView` off every capture response
before trusting the image. Both are cheap; neither is currently advised anywhere.

## Distinct from related tickets

- `B-set-game-view-does-not-suppress-spline-overlays` (OPEN, Medium) — the
  closest ticket, and a genuinely **different state**. There, game view is on and
  honest (`gameViewEnabled: true`, capture reports `gameView: true`) and overlays
  leak *through* it. Here game view was correctly enabled and then **turned off
  underneath the caller**, so `viewport.gameView` read `false` at capture time.
  That ticket frames the residual risk as the caller not reading the JSON — *"A
  caller who does not read `notGovernedByGameView[]` … is exactly as exposed as
  before"* — and this case defeats that framing, because the caller did read it
  and a third party undid the state afterwards. Its fix-shape option 2 ("have the
  **capture** verbs own it rather than `set_game_view`: a capture is a bounded
  operation with a natural restore point") is the right direction for this
  ticket too. And the `99eedf35` reasoning it quotes — that force-clearing would
  be "an unannounced global mutation it cannot restore" — is *strengthened* by
  this evidence (it really is a shared global), while showing that
  disclosure-without-a-warning is not sufficient.
- `B-game-view-reports-invariant-show-flags` (OPEN, Medium) — about four
  structurally-constant response fields on `set_game_view`. It supplies the
  `overlayWarning` precedent cited above but does not own the concurrency case or
  the capture-side warning gap.
- `B-set-game-view-keeps-editor-billboards` (IN-REVIEW, High) — `set_game_view`
  never actually toggled game view (a phantom `GEditor->Exec("ToggleGameView 1")`).
  Already fixed in-tree; this session confirms the toggle now works, since it
  reported the true `previous.gameViewEnabled: false`.
- `B-exposure-pin-black-frame` (IN-REVIEW, High) and
  `B-unlit-level-capture-no-warning` (OPEN, Medium) — same "capture returns a
  contaminated frame with no warning" *class*, but both about luminance and
  exposure, not chrome or shared state. Cross-reference, not duplicates.

severity rationale: impact=a capture presented as an acceptance shot silently contains editor chrome or an overlay a reviewer can read as content, with the invalidating fact measured in the response and never raised x reach=every level capture in any editor shared by more than one agent, which is the normal operating mode of this project -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Observed live on `/Game/Maps/Atlantis` while two placement agents were seating actors: a review capture taken with `hideEditorSprites: true`, in a burst that began with a confirmed `editor.set_game_view {enabled:true}`, came back with the world-axis gizmo drawn bottom-left and `viewport.gameView: false`, `meanLuminance` 0.331572; re-asserting game view and re-shooting the identical pose gave 0.343396 and no gizmo. That a third party moved the state is pinned by the next `set_game_view` reporting `previous.gameViewEnabled: false` with `previous.overlayShowFlags` showing `splines`, `grid`, `volumes`, `selection`, `selectionOutline` and `lightRadius` all back on, none of which the reporting agent had touched. `editorSprites` reported a clean `{hideRequested:true, visible:false, restored:true}` because the gizmo is not a billboard. Source-confirmed at HEAD in this tree during filing: `viewport.gameView` IS measured and published at `RenderHandler.cpp:1585` (`ViewportClient.IsInGameView()`) and `PreviewViewportCaptureUtils.cpp:2833`, while a plugin-wide grep for `gameViewWarning` and `requireGameView` returns **zero** hits — and the parallel warning pattern already ships twice, as `hideWarning` at `PreviewViewportCaptureUtils.cpp:2512` and `overlayWarning` at `ViewportHandler.cpp:706`. So the fix is a warning on a measurement the verb already takes. Worked around by re-asserting game view immediately before every capture and reading `viewport.gameView` off every response; defect untouched.
