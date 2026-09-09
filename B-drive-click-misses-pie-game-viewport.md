---
id: B-drive-click-misses-pie-game-viewport
title: "drive.click actuates nothing in a PIE session started from a level-based menu — two different buttons return no_change_within_budget / timeout with empty diffs and no log trace, while drive.observe sees both with valid hit-test geometry"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.click, pie, game-viewport, slate-injection, umg, no-change-within-budget, focus, dpi]
encounters: 1
costly: 1
lastSeen: 2026-09-09T10:00:00Z
---

# Clicks reach the observer but not the widget, in a PIE game viewport

UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`. PIE launched into the
level-based main menu (`L_Core`), whose UMG menu is drawn in the **game** viewport.

- `drive.observe` sees the menu correctly: `W_MultiplayerButton` and `W_OptionsButton`
  are both listed, with hit-test geometry.
- `drive.click` on `W_MultiplayerButton` returns `no_change_within_budget` / `timeout`
  with an **empty diff**.
- The same call on `W_OptionsButton`, used as a control because it opens a settings
  screen with an unmistakable visual change, behaves identically: no change, empty diff.
- **Nothing lands in the log** — no button handler trace, no navigation, no error.
- Driving the same widget by reflection *does* work: `ProcessEvent` on the button's
  `HandleButtonClicked`, and `ui.create_hud`, both produce the intended transition.

So the widget, its handler and the screen transition are all functional; only the
injected pointer event fails to actuate them, and it fails for two independent buttons on
the same screen.

## Not the settle-window ticket

`E-drive-click-no-change-on-slow-transition` (OPEN) is the case where the click
**succeeds** and the settle loop closes its 500 ms quiet window before the transition
becomes visible — there, the next `drive.observe` shows the destination screen. Here the
next observe shows the *same* screen, no navigation ever happens, and the reflection
route is what finally moves it. Same response string, opposite underlying outcome; a
fixer should not assume the settle budget explains this one.

Not `B-simulate-input-cef-click-noop` either: that is `editor.simulate_input`'s
malformed injection (null window, no effecting button) and is IN-REVIEW. `drive.click`
uses a different, better-formed path — `FDriveInput::ResolveNativeWindowUnder`
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Drive\DriveInput.cpp:142`)
feeding `SlateApp.ProcessMouseButtonDownEvent(NativeWindow, DownEvent)` at `:234` with
the press-handled bool captured, and `ProcessMouseButtonUpEvent` at `:239`.

## What needs diagnosing

The unknown is where the event is lost between that injection and the UMG button inside a
**PIE game viewport** (as opposed to an editor-chrome Slate widget, where `drive.click`
works). Candidates, in the order worth checking:

1. **Window resolution** — which `FGenericWindow` `ResolveNativeWindowUnder` returns for a
   PIE viewport, and whether it is the one Slate routes to.
2. **Focus / capture** — a PIE viewport that has taken mouse capture or set a game-input
   mode can swallow or re-route the synthetic event; the game viewport's input mode is
   not consulted anywhere on this path.
3. **Coordinate space** — screen vs window vs viewport coordinates, and DPI scaling: the
   observer's hit-test geometry and the injected screen position must be in the same
   space, and a wrong offset produces exactly this symptom (a click that lands on
   nothing, handled by no widget, with no error).

Report what the press-handled bool actually returns for this case — if it is `false`, the
response should say so rather than reporting a settle outcome, which is what makes the
failure indistinguishable from `E-drive-click-no-change-on-slow-transition`.

## Counter-evidence: a later real-click session did not reproduce it

A second real-click PIE verification on the same build and the same menu (UE 5.8,
`X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`, `L_Core` -> `W_Multiplayer` ->
`W_CreateMultiplayerRoom` popup, 2560x1440 at 150% Windows scaling) **could not reproduce
this bug**. There `drive.click` did actuate the game-viewport widgets:

- The first click returned `no_change_within_budget` — but it had landed; the CommonUI
  async layer push simply finished after the default settle budget.
- With an explicit `wait_for`, every subsequent click returned `wait_for_met` in
  **11-110 ms**.

That is `E-drive-click-no-change-on-slow-transition` exactly: a successful click whose
visible change begins after the budget, reported with the same outcome string as a true
no-op. So a bare `no_change_within_budget` is **not** by itself evidence that the click
missed, and a fixer must not read it as one.

What this does *not* settle is `#1`'s own case, which is why the ticket stays OPEN at
Medium rather than being re-scoped or downgraded: `#1` records a post-click
`drive.observe` showing the *same* screen (no navigation at all) plus a working reflection
route — i.e. it already applied the "look again after a longer wait" check and still saw
no effect. Either that observe was itself taken too early / against a stale surface, or
the two sessions differ in a way not yet identified. The next agent on this ticket should
re-run `#1`'s repro **with a `wait_for`** and either close it as the settle-budget false
negative or capture the press-handled bool that `#1` never recorded.

Related trap found in the same later session, worth ruling out before blaming input:
`drive.observe` reports `enabled:true, interactable:true` for a CommonUI button that is
actually disabled (`B-drive-observe-commonui-disabled-reads-enabled`). A click on such a
button legitimately does nothing while observe insists the target was live — this ticket's
symptom pattern from a different cause. `#1` used a disabled quick-host button as a
control, so it is directly in scope for the re-check.

**Workaround:** drive the widget by reflection — `object.call_function` / `ProcessEvent`
on the button's bound handler (`HandleButtonClicked`), or `ui.create_hud` for the target
screen. Both bypass input entirely, so neither exercises the real interaction and neither
proves the UI is reachable by a player.

## History
- `#1-two-buttons-no-actuation-in-pie` `OPEN` reporter — Filed from a PIE run on `L_Core`'s level-based main menu (UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`). `drive.observe` listed `W_MultiplayerButton` and `W_OptionsButton` with hit-test geometry; `drive.click` on each returned `no_change_within_budget` / `timeout` with empty diffs and left no log trace, and the screen never changed. Reflection-driving the same buttons (`ProcessEvent` on `HandleButtonClicked`) worked, which rules out the widget, its handler and the transition, and leaves the injected pointer event as the failing part. Deliberately filed separately from `E-drive-click-no-change-on-slow-transition` (whose click succeeds and whose next observe shows the destination screen) and from `B-simulate-input-cef-click-noop` (different verb, malformed injection, IN-REVIEW); the source citations above are the path to investigate, not a claimed root cause — no fix was attempted and the press-handled bool's value for this case was not captured. Marked `costly`: the whole menu-driving leg of the task had to be rebuilt on widget reflection, which does not test the input path at all.
- `#2-non-repro-with-wait-for` `OPEN` reporter — **Non-repro** on a later real-click PIE verification, same build and menu (plugin `fa755a4f`, `L_Core` -> `W_Multiplayer` -> `W_CreateMultiplayerRoom`, 2560x1440 at 150% Windows scaling): `drive.click` actuated the game-viewport widgets. The first click returned `no_change_within_budget` yet had landed — the CommonUI async layer push outran the default settle budget — and with an explicit `wait_for` every later click returned `wait_for_met` in 11-110 ms. Suggests `#1` may be the same false negative (`E-drive-click-no-change-on-slow-transition`), but the re-classification is **not** applied and severity is **not** lowered, because `#1` already reports a post-click `drive.observe` showing the same screen — i.e. no effect after a longer look — which is the exact gate for that call. Ticket stays OPEN at Medium pending a re-run of `#1`'s repro with `wait_for` and a captured press-handled bool. `encounters`/`lastSeen` deliberately not bumped: this entry records the bug NOT being observed, and counting it would inflate the work-ordering tiebreak. Also cross-referenced `B-drive-observe-commonui-disabled-reads-enabled` (filed from the same session): observe reports a disabled CommonUI button as `enabled:true, interactable:true`, which produces this ticket's symptom pattern from a different cause and must be ruled out during the re-check — `#1` used a disabled quick-host button as a control.
