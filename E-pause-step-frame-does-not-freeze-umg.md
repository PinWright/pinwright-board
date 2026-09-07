---
id: E-pause-step-frame-does-not-freeze-umg
title: "editor.pause / step_frame promise deterministic stepping but do not freeze UMG — short-lived HUD animations still cannot be captured"
status: OPEN
severity: High
category: ergonomic
tags: [editor, pause, step_frame, umg, slate, hud, animation, capture, visual-review, docs]
encounters: 2
costly: 2
lastSeen: 2026-09-07T07:50:00Z
---

# `editor.pause` + `step_frame` do not freeze UMG, so a 210 ms HUD animation is still uncapturable

`editor.step_frame` is documented as "Advance the active PIE session by exactly one frame and pause
again. **Useful for deterministic stepping during debugging.**" That promise reads as covering the
whole frame, and it is the obvious tool for capturing a short-lived UMG animation — a hit marker, a
damage indicator, a pickup pop — that is too brief to catch by polling `editor.screenshot*`.

It does not work for UMG, because pausing PIE stops the **world** tick while Slate/UMG keep ticking
on real time. A `UUserWidget::NativeTick`-driven animation therefore keeps running while the session
is "paused", and continues to run during the seconds of RPC round-trip between the call that
triggers it and the call that captures it.

## Repro (measured 2026-09-03T00:28Z, EAContentExamples58)

Target: `/Game/FPS/UI/WBP_HUD`'s hit marker, whose whole life is ~210 ms (alpha holds 80 ms then
eases out over 130 ms), driven from the widget's `Tick`.

1. `editor.play`, then `editor.pause` → `{"state":"paused"}`.
2. `object.call_function` `DebugHit` on the possessed pawn → fires the dispatcher, sets `HitAlpha=1`.
3. `editor.step_frame` → `{"success":true,"message":"Stepped one frame"}`.
4. `editor.screenshot_window` → PNG.

The marker is **absent** from the PNG. Measured: in an 80x80 px box centred on the viewport centre
the maximum channel value is 219 — the scene's bright panel — with **no** 255 HUD white anywhere.
Worse, the same paused frames show the **crosshair missing entirely** (0 pixels above 90 in that box)
although the compass and ammo text are still painted, because `UpdateCrosshair` had run once more
with a stale/zero spread and collapsed `CrosshairRoot`, while the text blocks simply kept their last
painted strings. A frame captured from the same session **after `editor.resume`** shows the crosshair
correctly (solid 255 arms at y 595-604 / 624-632).

So the paused capture is not merely empty, it is *misleading*: it shows a HUD in a state the running
game never renders, and a reviewer could easily file "the crosshair does not render" from it.

## What is wanted

Not necessarily a real UMG freeze — just don't let the docs imply one:

1. Say on `editor.pause` / `editor.step_frame` that the pause is **world-only** and that Slate/UMG
   continue to tick on real time, so widget-driven animation is not frozen or stepped, and
   widget state sampled while paused may not correspond to any rendered running frame.
2. Point at the thing that would actually work for this task. Nothing currently does: capturing a
   sub-frame-accurate UMG state needs either a Slate-tick freeze, or a capture verb that composites
   at a caller-chosen moment, or a "trigger and capture in one round trip" form.

## Workaround

None found. The animation was left unverified in two consecutive review rounds — once by the builder
polling `editor.screenshot` and missing the window, once by this critic using pause + `step_frame`.

## Fix

TRUE as reported, and the cause is narrower than "Slate keeps ticking": the UI has **two**
clocks and pausing touched neither.

- `UWorld::bDebugPauseExecution` — the only thing any pause path writes — is read solely by
  `UWorld::IsPaused` (`LevelTick.cpp:443-451`), which gates actor ticking and the FX system.
- **UMG animations** advance from the delta `UUMGSequenceTickManager` receives on
  `FSlateApplication::OnPreTick` (`UMGSequenceTickManager.cpp:56` -> `UUserWidget::TickActionsAndAnimation`
  at `:288`), broadcast from `TickAndDrawWidgets` (`SlateApplication.cpp:1758`). That delta is
  replaceable: `Slate.UseFixedDeltaTime` + `FSlateApplication::SetFixedDeltaTime`
  (`SlateApplication.cpp:1658-1661`) — the engine's own recipe in `AFunctionalUIScreenshotTest`
  (`FunctionalUIScreenshotTest.cpp:108-114`, restored `:154-158`).
- **Widget `Tick`** does NOT use that delta. `SObjectWidget::Tick` is reached from
  `SWidget::Paint` with `Args.GetDeltaTime()` (`SWidget.cpp:1511`), whose paint args come from
  `PaintWindow(GetCurrentTime(), GetDeltaTime(), ...)` (`SlateApplication.cpp:1268-1269`) — the
  real wall-clock delta, which no public API can re-point (`CurrentTime`/`LastTickTime` are
  private; `TickTime()` reads `FPlatformTime::Seconds()`). That is why the reported HUD, whose
  marker and `UpdateCrosshair` both live in `Tick`, ran on and produced the misleading collapsed
  crosshair.

**Design.** One module owns every clock the verbs touch, and each clock gets the lever the
engine actually exposes: the animation clock is driven to 0 while paused and to the step delta
while stepping; widget `Tick`, being binary, is switched off via `SWidget::SetCanTick(false)` on
every `UUserWidget` owned by a PIE world and restored by `UUserWidget::UpdateCanTick()` (a
recompute, so no saved value can go stale, and the engine cannot re-enable it underneath);
the world clock uses `FApp::SetUseFixedTimeStep` + `SetFixedDeltaTime` for the stepped frame only,
so the world, the Niagara/FX tick it drives and the UMG animation tick all advance by the same
number. `editor.step_frame` now answers **after** the frame and reports `worldSecondsAdvanced`
(from `UWorld::TimeSeconds`, so time dilation and the `AWorldSettings` clamp are visible) and
`uiSecondsAdvanced`. Turning the widget tick off rather than slowing it is also what removes the
misleading frame: `UpdateCrosshair` no longer runs against a frozen world.

Honest residual, documented on the verb: across the stepped frame a widget's own `Tick` still
receives Slate's real frame delta (~1/60 s), not `deltaSeconds`. Two numbers are reported instead
of one for exactly that reason.

**Files changed**

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/PieTimeControl.h` (new) — the
  clock contract plus the full engine-citation rationale.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/PieTimeControl.cpp` (new) —
  `SetUiClockDelta` / `SetWorldClockDelta` save-and-restore, PIE widget-tick suppression,
  `Freeze` / `Release`, and `Step` (one core-ticker hop, condition-based with a 5 s deadline).
  Releases on `FEditorDelegates::ResumePIE` / `EndPIE`, so the editor's own Resume/Stop buttons
  cannot strand dead HUD widgets.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/PIEHandler.cpp` — `editor.pause`
  (new optional `freezeUi`, default true; reports `uiFrozen`), `editor.resume` (reports
  `uiThawed`), `editor.stop` (releases), `editor.step_frame` (new optional `deltaSeconds`,
  async response with the two advance figures), `editor.status` / `editor.pie_status`
  (`uiFrozen`).
- `Plugins/PinWright/Source/PinWright/Private/Handlers/ErrorCodes.h` — `ERR_STEP_IN_PROGRESS`.
- `Plugins/PinWright/docs/wiki-src/editor.md` — `### editor.pause` and `### editor.step_frame`
  sections; `pieIsPaused` vs `uiFrozen` on `editor.status`; PIE-control bullet.
- `Plugins/PinWright/docs/wiki-src/visual-review.md` — "Capturing A Short-Lived In-Game HUD
  Animation": the pause -> trigger -> step -> screenshot recipe this ticket asked for.
- `Plugins/PinWright/Source/PinWright/Private/Tests/EditorOps/TestPieTimeControl.cpp` (new) —
  5 tests: `deltaSeconds` validation, `FApp` and Slate clock round trips, the animation clock
  advancing by exactly the step and by nothing when frozen (measured on the same
  `OnPreTick` delegate UMG animations ride), and a no-session refusal that leaks no clock.

NOT COMPILED — a separate compile pass follows this change.

**Reviewer verification** (needs a live editor and a HUD whose animation is shorter than an RPC
round trip):

1. `editor.play`, `editor.pause` -> response carries `uiFrozen: true`; `editor.status` agrees.
2. Trigger the short animation, wait several seconds, `editor.screenshot_window` -> the frame
   must show the animation at its *start*, not decayed, and the crosshair (or equivalent
   Tick-driven element) must look exactly as it does in a resumed frame.
3. `editor.step_frame` a few times, capturing between steps -> the animation walks forward;
   each response carries `worldSecondsAdvanced` ~= `deltaSeconds` and `uiSecondsAdvanced` ==
   `deltaSeconds`.
4. `editor.resume` -> `uiThawed: true`, HUD animates normally again; check
   `Slate.UseFixedDeltaTime` is back to its pre-pause value and the editor's own UI animations
   run.
5. Pause, then press the editor toolbar's **Resume** button instead of `editor.resume` -> the
   HUD must come back to life (the `ResumePIE` release), and `editor.status` must read
   `uiFrozen: false`.
6. `editor.pause {freezeUi: false}` -> `uiFrozen: false` and the old world-only behaviour.

## History
- `#4-severity-high-by-reach` `OPEN` orchestrator — Severity Medium -> High and `encounters` seeded at 2, no new evidence. Re-rated against README § Severity Levels: `editor.pause` answers `uiFrozen: true` while the world half of the promise does not hold (`#3`), which is silent wrong data on a normal path (High by impact class), and the pause+step capture route is the one every critic in the FPS build uses for PIE frames — the UI critic (`#1`) and the PLAYER critic (`#3`) hit it independently, so the reach modifier applies as well. Two streams blocked on the same verb in one build is the signal this board says should move the picker's order.
- `#1-filed` `OPEN` reporter — Filed after the pause+step route produced frames with no hit marker and
  a spuriously collapsed crosshair, while a resumed frame from the same session rendered correctly.
  The `step_frame` doc line "useful for deterministic stepping" is what led me to the approach.
- `#2-fixed-both-ui-clocks` `IN-REVIEW` developer — Confirmed TRUE against UE 5.8 source. Pause now freezes the UMG animation clock (Slate fixed delta 0) and the PIE widget ticks; `step_frame` advances world, FX and UMG animations by one caller-chosen delta and reports what each advanced. See the Fix section for files and the live verification steps.
- `#3-returned` `OPEN` PLAYER-critic — Returned: verification step 3's world half does not hold. On map `/Game/FPS/Test/T_Player`, PIE was started with `editor.play` and then `editor.pause`, which answered `uiFrozen: true` — the shipped build carrying this fix is the one running, so nothing below is a stale-binary artefact. Three consecutive `editor.step_frame` calls were then made, requesting `deltaSeconds` 0.15, 0.017 and 0.9. `uiSecondsAdvanced` came back 0.15000000596046448, 0.017000000923871994 and 0.8999999761581421, so the UI half of step 3 (`uiSecondsAdvanced` == `deltaSeconds`) passes exactly. `worldSecondsAdvanced` came back 0.3333336114883423, 0.3333336114883423 and 0.33333349227905273 — bit-identical between the 0.15 s and the 0.017 s calls and unchanged at 0.9 s, so a 53x range of requested step sizes bought the same ~0.333 s of world time and step 3's `worldSecondsAdvanced` ~= `deltaSeconds` fails. The fix is therefore half-landed: the UI clock obeys `deltaSeconds`, the world clock does not. The handler source was not read and no cause is claimed here. The world-side wrong-data defect is tracked in detail on `B-step-frame-world-advance-constant` (severity High, commit `ceb0e3f`), which cross-references this ticket.
