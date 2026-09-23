---
id: B-drive-click-misses-pie-game-viewport
title: "drive.click actuates nothing in a PIE session started from a level-based menu — two different buttons return no_change_within_budget / timeout with empty diffs and no log trace, while drive.observe sees both with valid hit-test geometry"
status: OPEN
severity: High
category: bug
tags: [drive, drive.click, drive.hover, simulate_input, pie, pie-in-viewport, game-viewport, slate-injection, mouse-capture, commonui, umg, no-change-within-budget, focus, dpi, coordinate-space]
encounters: 3
costly: 3
lastSeen: 2026-09-23T20:55:00Z
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
- `#3-repro-with-pie-hosted-in-the-level-viewport` `OPEN` reporter — **Reproduced**, 2026-09-10, host `X:\src\unreal\unreal-fpv-new`, UE 5.8, plugin `d5dfb11b`, and it carries the distinguishing fact `#1` and `#2` both lack: **PIE was running inside the level-editor viewport, with no standalone PIE window**. Nothing on the game surface actuated — `drive.click`, `drive.hover` and `editor.simulate_input {type:"mouse_click", target:"game"}` all left frontend buttons, the HUD menu button and the settings tabs inert; `drive.observe` reported `focused:false` for **every** element; `drive.click` returned the benign `no_change_within_budget`; reported geometry sat about (+37,+254) px from the same widgets in captured viewport pixels. Keyboard injection on the same session worked (`editor.simulate_input key_down/key_up target:"game"` → `deliveredToGame:true`), so PIE was live and receiving input. Four source findings against HEAD, none of them fixed. **(a) The press-handled bool `#1` asked for is one call away.** `FDriveInput::ClickAt` (`DriveInput.cpp:196-203`) explicitly discards it (`bDiscardHandled`) and returns true whenever Slate is initialised; `RunAction` errors only when the lambda returns false (`DriveActionCommon.cpp:253-258`), and the response object (`:274-296`) has no `handled` field — so "Slate was up" is the entire success test, which is exactly what makes this indistinguishable from `E-drive-click-no-change-on-slow-transition`. `ClickAtReportingHandled` already exists (`DriveInput.cpp:205-242`) and `editor.simulate_input` already uses it honestly (`EditorCommandHandler.cpp:929-941`); `drive.click` simply does not call it. **(b) The coordinate-space suspect is now resolved and is NOT the cause of the observed offset.** Element geometry is window-space — `DriveLiveResolver.cpp:342-344` reads `GetCachedGeometry().GetAbsolutePosition()`, and Slate roots cached geometry at `SWindow::GetWindowGeometryInWindow()` (`SWindow.cpp:2143`) — so (+37,+254) is just the level viewport's origin inside the editor frame (toolbar strip plus menu/toolbar/tab-bar), expected against a viewport-local screenshot. The real gap is that this window-space value is injected as **desktop** pixels (`DriveInput.cpp:125` `SetPosition`, `:146` `LocateWindowUnderMouse`, `:234` `ProcessMouseButtonDownEvent`) with no `SWindow::GetPositionInScreen()` term anywhere, which is only harmless while the owning window's client origin is (0,0) — filed separately as `B-drive-geometry-window-space-injected-as-desktop`. **(c) New leading suspect, and it is coordinate-independent.** `FSlateApplication::ProcessMouseButtonDownEvent` (`SlateApplication.cpp:5335-5377`) tests `SlateUser->HasCapture(pointerIndex)` *first* and, when a capture exists, routes only down the captor path — `LocateWindowUnderMouse` is never called. A PIE viewport holding mouse capture therefore swallows every synthetic press wherever it points, with no error and no reaction, which is precisely this ticket's symptom set. It also explains the `focused:false` on every element (`DriveLiveResolver.cpp:340`, `HasAnyUserFocus()`): focus sits on the `SViewport`, not on the CommonUI buttons. The drive path never queries or releases capture and never consults the viewport's input mode; grep found no `SLevelViewport` / `IsPlayInEditorViewport` / `bIsPlayInEditor` handling in it at all, so PIE-in-viewport and standalone-PIE are treated identically — a plausible reason `#2` (which did actuate) and this run differ. **(d) `target:"game"` did nothing on the mouse branch.** `editor.simulate_input`'s `target` is documented "Ignored by mouse events" (`EditorCommandHandler.cpp:752`) and is never read there, so that call was plain Slate injection. There is no `FDriveGameInput` mouse route at all — which is exactly why keyboard works: it bypasses coordinates and hit-testing entirely, delivering through `UGameViewportClient::InputKey` on the world's post-actor-tick (`DriveGameInput.cpp:199-213`) with `deliveredToGame` measured against the target's own `UPlayerInput` event ids (`:226-239`). Also rule out `B-drive-observe-collapsed-ancestor-reads-visible` (filed the same day): observe's geometry is `GetCachedGeometry()`, which a non-painted subtree never updates, so a stale rect can aim a click at coordinates nothing occupies. Confirmed unfixed at HEAD `d5dfb11b` — `DriveLiveResolver.cpp`, `DriveActionCommon.cpp` and `DriveSettleDecision.cpp` have had no functional change since `b3ddbe55`, and the only coordinate-adjacent commit is `e08ff518` (`B-simulate-input-cef-click-noop`). No fix attempted. Marked `costly`: the UI-driving leg fell back to keyboard and reflection again, so the input path went untested for a second task. `encounters` 1→2 and `costly` 1→2; severity deliberately left at Medium, since the cost modifier bumps only at `costly` 3.
- `#4-one-cause-occluding-editor-window` `OPEN` reporter - UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, PIE in the level viewport on `L_Core`. **One concrete cause found.** `drive.click` on `W_ForcedLogoutPopup_C_0/OkButton/SCommonButton` and on `W_LoginOverlay_C_1/W_Login/BackButton/SCommonButton` returned `no_change_within_budget` with empty diffs; `drive.input_state` showed the OS cursor exactly at the button centre (1136,736) and `focused_widget` `SDockingTabStack`. `drive.list_windows` then showed a second top-level window, **`Message Log` at (0,0,1367x807)**, that the editor had opened on PIE start and that covered the click point. After closing its tab with `drive.click {surface: editor_chrome}` every game-surface click worked for the rest of the session (about 40 clicks). So the injected click is hit-tested against whatever top-level window is at that desktop point, and the verb neither detects nor reports that the point belongs to a different window. Proposed fix: before injecting, hit-test the desktop point (`FSlateApplication::LocateWindowUnderMouse`) and refuse with `TARGET_OCCLUDED` naming the occluding window title. Costly: about 10 calls and a Python workaround before the cause was found.
- `#5-bumped-by-cost` `OPEN` reporter - Severity Medium -> High under the cost rule: `costly` reached 3 from independent tasks (`#1` unreal-fpv-dev menu pass, `#3` unreal-fpv-new UMG pass, `#4` unreal-fpv-new login regression pass).
