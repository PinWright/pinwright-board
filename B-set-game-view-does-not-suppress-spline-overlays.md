---
id: B-set-game-view-does-not-suppress-spline-overlays
title: "set_game_view reports a clean frame but does not suppress spline overlays — a water spline's white dashed bank lines render into captures and have twice been on the verge of being reported as map content"
status: OPEN
severity: Medium
category: bug
tags: [editor, viewport, set_game_view, capture, overlays, splines, water, false-feature, review-hazard, shared-state, concurrency, multi-agent]
encounters: 2
lastSeen: 2026-08-27T18:47:15+05:00
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

## Encounter 2026-08-27 — a second state this ticket's framing does not cover: the caller got it right and a concurrent agent undid it

Found building the Atlantis level on UE 5.8 (PinWright at the EAContentExamples58
checkout's HEAD), in an editor shared by several agents. **This does not
contradict anything above — it is a different state, and it defeats one line of
this ticket's reasoning.**

This ticket's whole frame is single-caller: game view is on and *honest*
(`gameViewEnabled: true`, capture reports `gameView: true`) and overlays leak
through it anyway. The state observed here is the opposite one. A review capture
taken with `hideEditorSprites: true`, in a burst that began with a confirmed
`editor.set_game_view {enabled: true}`, came back with the **world-axis gizmo**
drawn bottom-left, and the response's own `viewport.gameView` read **`false`**.
Game view had been turned off underneath the caller between one call and the
next. `editorSprites` reported a clean `{hideRequested: true, visible: false,
restored: true}`, because `hideEditorSprites` did its job on billboards and the
gizmo is not a billboard.

Measured: `meanLuminance` 0.331572 with the gizmo; re-asserting game view and
re-shooting the identical pose gave 0.343396 and no gizmo. That a third party
moved the state is pinned by the next `editor.set_game_view {enabled: true}`
reporting `previous.gameViewEnabled: false` with `previous.overlayShowFlags`
showing `splines`, `grid`, `volumes`, `selection`, `selectionOutline` and
`lightRadius` **all back on** — the full editor set, none of which the reporting
agent had touched.

**Which line this narrows.** This ticket states the residual risk as the caller's
omission: *"A caller who does not read `notGovernedByGameView[]`, or an
image-diffing pass that never sees the JSON at all, is exactly as exposed as
before."* In a shared editor that is not the only exposure. The caller here
**did** assert game view and **did** get a clean confirmation; a concurrent agent
undid it afterwards. Disclosure in the JSON cannot cover a value that was true
when disclosed and false when the shutter fired. Game view is per-**viewport**
state, so in a multi-agent editor it is a global any agent can move under any
other.

**Which line this strengthens.** The `99eedf35` reasoning quoted here — that
force-clearing the flag would be "an unannounced global mutation it cannot
restore" — is *more* right than it looked: it really is a shared global with
other owners. And this ticket's own fix-shape option 2 is the right direction for
both states: *"Have the capture verbs own it rather than `set_game_view`: a
capture is a bounded operation with a natural restore point, which
`set_game_view` (a persistent mode toggle) is not."*

The concurrency case is filed separately as
`B-game-view-shared-state-no-capture-warning`, because its fix is a missing
`gameViewWarning` / `requireGameView` on the capture verbs rather than anything
about spline show-flags. Source note relevant to both: `viewport.gameView` is
already measured and published at `Handlers/Render/RenderHandler.cpp:1585`
(`ViewportClient.IsInGameView()`) and
`Handlers/Render/PreviewViewportCaptureUtils.cpp:2833`, while a plugin-wide grep
for `gameViewWarning` and `requireGameView` returns zero hits — and this ticket's
own `overlayWarning` precedent lives at `Handlers/Editor/ViewportHandler.cpp:706`.

## History
- `#1-warned-not-suppressed` `OPEN` reporter — `editor.set_game_view` reported `gameViewEnabled: true` and the capture reported `gameView: true`, yet a water spline still rendered a white dashed line down each bank into the frame (`tc_probe_centre.png`). Recorded in the host project at `Docs/map/reference_tile_compare.md:236-239`, which notes this is **"the exact artifact a previous reviewer nearly logged as foam"** — i.e. an editor overlay twice on the verge of being reported as authored map content, because a review pass reading a rendered frame cannot distinguish overlay from geometry and the verb's success response implied there was nothing left to distinguish. **Half of this is already fixed and the split matters.** `99eedf35` (on `origin/master`) closed the honesty half: `Handlers/Editor/ViewportHandler.cpp` reads back ten overlay show flags off `EngineShowFlags` and emits `notGovernedByGameView[]` unconditionally (`:706`), so the response can no longer read as a blanket clean-frame claim, with `Tests/EditorOps/TestSetGameViewOverlayReporting.cpp` (157 lines) asserting the array's presence at `:137-138`. The suppression half is what remains, and the commit states why it was left: *"The verb does not force-clear the flag; that would be an unannounced global mutation it cannot restore."* That reasoning is right — silently clearing a global show flag with no restore is worse than disclosing it — but the hazard is unchanged for anyone who does not read the JSON, including any image-diffing pass that never sees it. Fix shape, cheapest first: an opt-in `suppressOverlays`-style parameter that clears the named flags for one call and restores them (the restore obligation is takeable explicitly even though it was rightly refused implicitly); or move the responsibility to the capture verbs, which are bounded operations with a natural restore point where `set_game_view` is a persistent mode toggle; or, if neither, say it plainly on the wiki page in the caller's language so a reviewer reading a PNG is warned where they actually look.
- `#2-additional-concurrent-agent-clears-game-view` `OPEN` reporter — Additional evidence (Atlantis level build, UE 5.8, EAContentExamples58 checkout HEAD, editor shared by several agents). Adds a SECOND state this ticket's single-caller framing does not cover, and narrows one line of its reasoning; see the dated encounter section above for the full account. Summary: a review capture taken with `hideEditorSprites: true`, in a burst opened by a confirmed `editor.set_game_view {enabled: true}`, returned the world-axis gizmo in frame with the response's own `viewport.gameView` reading **false** — game view was cleared by a concurrent agent between the assert and the shutter. `meanLuminance` 0.331572 with chrome vs 0.343396 on an identical re-shot pose after re-asserting. Third-party attribution pinned by the next `set_game_view` reporting `previous.gameViewEnabled: false` with `previous.overlayShowFlags` showing `splines`/`grid`/`volumes`/`selection`/`selectionOutline`/`lightRadius` all restored, none touched by the reporting agent. `editorSprites` reported clean (`hideRequested:true, visible:false, restored:true`) because the gizmo is not a billboard. NARROWS this ticket's stated residual risk ("A caller who does not read `notGovernedByGameView[]` ... is exactly as exposed as before"): the caller here read it and got a clean answer, and the state changed afterwards — JSON disclosure cannot cover a value that was true when disclosed and false when the frame was drawn. STRENGTHENS the `99eedf35` reasoning that force-clearing would be an unrestorable global mutation (it really is a shared global with other owners), and supports this ticket's own fix-shape option 2 (capture verbs own the flag, since a capture is bounded and `set_game_view` is a persistent mode toggle). Concurrency case filed separately as `B-game-view-shared-state-no-capture-warning` because its fix is a missing `gameViewWarning` / `requireGameView` on the capture verbs, not spline show-flag behaviour. Source cross-check at HEAD: `viewport.gameView` already measured/published at `RenderHandler.cpp:1585` and `PreviewViewportCaptureUtils.cpp:2833`; plugin-wide grep for `gameViewWarning`/`requireGameView` returns zero hits; the `overlayWarning` precedent is at `ViewportHandler.cpp:706`. No new bug; widens confirmed scope to concurrent-agent state loss.
