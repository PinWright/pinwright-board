---
id: B-simulate-input-cef-click-noop
title: "editor.simulate_input mouse_click injects a malformed Slate event (nullptr window, no effecting button) and hardcodes success — clicks silently miss the widget yet report {success:true}"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, simulate_input, mouse_click, slate-injection, effecting-button, silent-false-success, cef, swebbrowser]
encounters: 3
lastSeen: 2026-09-30T16:48:52Z
---

# editor.simulate_input mouse_click is a malformed Slate injection with hardcoded success

`editor.simulate_input {type:"mouse_click", x, y}` builds its Slate pointer events
incorrectly and then reports `{success:true}` unconditionally. Two defects, both on
the ordinary path (any coordinate click, not only over CEF):

1. **Broken injection.** The `mouse_click` branch passes `nullptr` as the platform
   window to `ProcessMouseButtonDownEvent`, builds `FPointerEvent`s via the
   move/delta constructor with **no effecting button** (`EKeys::Invalid`), sets no
   inactive-input flag, and the mouse-**up** event carries an **empty** released-
   buttons set. A press with no effecting button + a null window frequently fails
   to route to the widget under the cursor, so the click actuates nothing.
2. **Silent false-success.** `bSuccess = true` is hardcoded, discarding the handled-
   bool returns of `ProcessMouseButtonDownEvent`/`ProcessMouseButtonUpEvent`. A
   caller who clicked nothing is told the click landed — a lie it then builds on.

The bug was first observed clicking a `.pdd` control inside an embedded CEF
`SWebBrowserView` (the DOM handler never fired, yet the call reported success), but
the root cause is **not** "Slate injection can't reach CEF." The engine's
`SWebBrowserView` already forwards a routed Slate mouse event into the browser
window/CEF DOM through its own `SViewport`/`FWebBrowserViewport` — so a
**correctly** routed click reaches CEF for free. The click failed because the
injection was malformed, not because CEF forwarding is missing.

## Guilty source
`Handlers/Editor/EditorCommandHandler.cpp` — `editor.simulate_input` registered at
line 695; the pre-fix `mouse_click` branch built two `FPointerEvent`s with a
`nullptr` window and no effecting button, routed them via
`SlateApp.ProcessMouseButtonDownEvent(nullptr, …)` /
`ProcessMouseButtonUpEvent(…)`, and set `bSuccess = true` unconditionally. Contrast
the plugin's own vetted coordinate-click primitive `FDriveInput::ClickAt`
(`Handlers/Drive/DriveInput.cpp:187`), which resolves the native window under the
point (`ResolveNativeWindowUnder`), enables `SetHandleDeviceInputWhenApplicationNotActive`
(its in-code comment: without it "the button-up never fires the click"), dispatches
a mouse-move first, and builds a full `FPointerEvent` **with** the effecting button.
`editor.simulate_input` did none of this.

## Session evidence (replayable)
Live PIE, CEF physics panel. `PhysicsWebBrowser`/`SWebBrowserView` rect (from
`drive.observe(game)`): x=1761 y=488 w=673 h=523. Target `.pdd` dropdown ("4S 1550mAh
100C") DOM geometry (from `drive.observe(web)`): x=2268 y=761 w=113 -> click center
~(2324,772), inside the browser rect.
1. `editor.simulate_input {type:"mouse_move", x:2324, y:772}` -> `{success:true}`.
2. `editor.simulate_input {type:"mouse_click", x:2324, y:772, button:"left"}`
   -> `{success:true, message:"Mouse click at (2324.000000, 772.000000)"}`.
3. Screenshot immediately after: panel UNCHANGED, dropdown did not open — the click
   never actuated. Contrast: `drive.click` (which routes through `FDriveInput::ClickAt`)
   reliably actuated exposed handles in the same panel.

**Fix:** Route the `mouse_click` branch through the vetted `FDriveInput` click
primitive (native-window resolution + inactive-input flag + full effecting-button
`FPointerEvent` + move-first) so synthesized clicks actually reach the widget under
the point — including SViewport-hosted CEF browsers, which forward the routed Slate
click into the DOM themselves. Report success **honestly** from whether a widget
consumed the press (the `Process*` handled-bool), instead of the hardcoded
`bSuccess = true`. Scope: `mouse_click` only — the `key_down`/`key_up` branches
discard their handled-bool too, but a key that no focused widget consumes is a
routine, legitimate injection, so honoring that bool would flip many valid key
injections to false; left as-is deliberately.

Related (cross-ref): `B-drive-web-omits-custom-interactables` (a `drive.observe`
interactable-detection gap on a different file/RPC — DriveWebBridge.cpp),
`B-drive-web-handle-collision` (duplicate `pw-N` handle minting). Both are distinct
code paths, not duplicates of this handler defect.

severity rationale: impact=silent-false-success — a click that actuated nothing (or
misrouted) returned affirmative success the caller builds on (the board's High
silent-false-success class) — AND the injection itself was malformed, so
coordinate-based clicks broadly under-actuate, not only over CEF. Reach=`editor.simulate_input`
is the general-purpose coordinate-click primitive used for menus, toolbars,
viewports, and CEF panels alike — a broad, shipped surface. Both point to High. (The
earlier "no workaround for handle-less custom CEF controls" line conflated this with
the sibling detection ticket `B-drive-web-omits-custom-interactables`; the intrinsic
silent-false-success + broken-injection defect justifies High on its own.)

## History
- `#1-cef-click-noop` `OPEN` reporter — Filed: `editor.simulate_input {mouse_click}` at coords over a live CEF `SWebBrowserView` (PDS physics panel) returned `{success:true, message:"Mouse click at (2324,772)"}` but the post-click screenshot showed the panel unchanged — the DOM click handler never fired. Confirmed against source: `EditorCommandHandler.cpp:745-774` builds `FPointerEvent`s and routes them through `SlateApp.ProcessMouseButtonDownEvent(nullptr,...)` / `ProcessMouseButtonUpEvent(...)` (765,771) with zero CEF/`SWebBrowserView`/`IWebBrowserWindow` forwarding, and hardcodes `bSuccess=true` at line 773 (discarding the `Process*` handled-bool) — a silent false-success. No coordinate-based path reaches CEF DOM; `drive.click` (DOM injection) works but only for observer-exposed handles, so handle-less custom `.pdd` controls are unclickable by any means. No existing board file mentions `simulate_input` or CEF/SWebBrowser click (ripgrep clean). Note: named sibling tickets `F-drive-web-omits-custom-interactables` / `B-drive-web-handle-collision` are not yet on the board (likely filed concurrently).
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD + fix. Verified against synced plugin HEAD: the primary "no CEF forwarding" root cause was WRONG — the engine's `SWebBrowserView` forwards a routed Slate click into CEF via its own `SViewport`/`FWebBrowserViewport` (SWebBrowserView.cpp), and the plugin's own `FDriveInput::ClickAt` (DriveInput.cpp:187-219) proves correct Slate injection reaches widgets. The real defect is a general **malformed injection + hardcoded silent-success**: the `mouse_click` branch passed a `nullptr` window and an effecting-button-less `FPointerEvent`, then set `bSuccess=true` regardless (EditorCommandHandler.cpp:762-773). Reworded title/body/Guilty-source/Fix/severity-rationale off the speculative CEF-forwarding framing and onto the verified defect; corrected the stale `F-` cross-ref prefix to `B-drive-web-omits-custom-interactables`. Fix: added `FDriveInput::ClickAtReportingHandled` (DriveInput.h/.cpp) which performs the vetted ClickAt injection (native-window resolution + inactive-input flag + full effecting-button `FPointerEvent` + move-first) and returns whether a widget consumed the press; `ClickAt` now delegates to it (its `true`=injected contract preserved for `drive.click`). Rewired `editor.simulate_input` `mouse_click` (EditorCommandHandler.cpp) to route through it and report success honestly (no widget consumed -> not success), instead of the hardcoded `bSuccess=true`. Scoped to `mouse_click` (key branches left as-is on purpose — honoring their handled-bool would false-negative routine key injections). Files: `Source/PinWright/Private/Handlers/Drive/DriveInput.h`, `Source/PinWright/Private/Handlers/Drive/DriveInput.cpp`, `Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp`. Regression test `PinWright.editor.simulate_input.MouseClickHonestSuccess` (TestEditorHandlers.cpp): drives the production handler against an in-code top-level SWindow+SButton, clicking the button's screen-space center and asserting its OnClicked fired AND the handler reported success — fails under the reverted nullptr-window / effecting-button-less injection (OnClicked never fires) and under a hardcoded-false over-correction (success assertion). Compiled clean and the test was executed headless (-RenderOffScreen -nocefaccelpaint) to Result={Success}. (An empty-space "not handled -> success:false" counterfactual was tried but proved unreliable headless: Slate's ProcessMouseButtonDownEvent returned handled over a window-less point under -RenderOffScreen mouse capture — the same imperfect-signal caveat the review lenses raised — so the guard is anchored on the deterministic routed-click case.)
- `#3-second-false-success-stale-routing` `IN-REVIEW` developer — The `#2` fix still reported false success. Regression test `PinWright.editor.simulate_input.MouseClickHonestSuccess` failed on this Linux host in the offscreen full run `Saved/PinWright/test-runs/0a3aa810dba14743b7c47ef45fb4e4e2` and in the baseline runs `integrator_base_fails` and `integrator_base_si`, and passed in run `693f8d20ebbd4e59979c3e44d76cfdbd`. Every failure raised only the `OnClicked` assertion, never the success assertion, so the handler said success for a click that reached no button. Two causes. (1) `FSlateApplication::ProcessMouseButtonDownEvent` returns `true` on every path (SlateApplication.cpp:5292-5404), so the `#2` "handled" signal from `ClickAtReportingHandled` was a hardcoded true under another name. That also explains why the `#2` empty-space counterfactual "proved unreliable". (2) Slate hit-tests the press inside the window that `PlatformApplication->GetWindowUnderCursor()` names (`LocateWindowUnderMouse`, SlateApplication.cpp:1151). On Linux that is `FLinuxApplication::CurrentUnderCursorWindow`, which changes only when an SDL `MOUSE_ENTER`/`MOUSE_LEAVE` event is pumped in the engine loop. A plain `FSlateApplication::Tick` does not pump it (TickPlatform pumps only under a modal window). So in the frame that shows a new window, the cache still names an older window. Under `-RenderOffscreen` (the SDL dummy driver) the only thing that moves SDL mouse focus is `SDL_ShowWindow`, because the cursor warp stays inside the current focus window (SDL_video.c:3413, SDL_mouse.c:1330). The click went to whichever window the cache named. The same run showed the same state in `drive.click_occlusion.UncoveredTargetIsClicked`, which was skipped with "sits under host window ''" while this test failed. In the passing run both tests passed. **Fix:** new `FDriveInput::TopWindowAtPoint` (DriveInput.h/.cpp) finds the top interactive window at the point by Slate's own window order, without the platform hint. It is the fallback half of `LocateWindowUnderMouse`, with `LocateWidgetInWindow`'s protected test rewritten against the public hit-test grid. The `editor.simulate_input` `mouse_click` branch (EditorCommandHandler.cpp) now calls `MoveTo`, then `ClickAt` only once `WindowUnderPoint` equals `TopWindowAtPoint`. It checks at once, then polls on the core ticker for up to 0.3 s so the platform can catch up (the same budget as the drive route gate). Otherwise it fails with `INPUT_FAILED` and data `{x, y, window, routedWindow}` and clicks nothing. It also fails with `INPUT_FAILED` when no window is at the point. A success carries `window`. Removed `FDriveInput::ClickAtReportingHandled`: its handled bool was always true and this handler was its only caller. `ClickAt` is otherwise unchanged. Wiki: `docs/wiki-src/editor.md` § editor.simulate_input. **Test:** `MouseClickHonestSuccess` now waits for the answer across real frames (`InvokeHandlerWithSharedCapture` + `FFunctionLatentCommand`). A success must come with `OnClicked` fired; the unfixed handler failed exactly this whenever the cache was stale. A `routedWindow` refusal asserts that nothing was clicked and emits `PINWRIGHT_ASSERTIONS_SKIPPED reason=platform-cursor-window-stale` instead of passing silently. Compile-checked with `-SingleFile`: DriveInput.cpp, EditorCommandHandler.cpp and TestEditorHandlers.cpp all succeeded. check_test_ids and check_test_skips are CLEAN. **Needs a live run:** `editor_run_tests` filter `PinWright.editor.simulate_input+PinWright.drive.click_occlusion`, mode `offscreen`. The null-rhi guard skips the test under headless. An offscreen run should end in pass or in the honest skip, never in a failure. A pass is not guaranteed. In run 0a3aa810 the drive route gate kept seeing the stale untitled window for its whole 0.3 s wait after showing its own fixture, so in that session state this test will most likely skip with `platform-cursor-window-stale`. Why the dummy-driver cache stays on that window is tracked in `B-offscreen-stale-cursor-window`.
- `#4-offscreen-cause-notification-window` `IN-REVIEW` developer — The `#3` build (4966b353) still failed offscreen in run 16964ca9. The handler took the success branch, so its routing matched the top window, yet `OnClicked` did not fire. Temporary debug logging in run 3acc89ff found the real offscreen cause. The offscreen virtual display is 640x360. The editor's notification window is untitled, topmost and holds the toasts; it covered the fixture. The widget leaf at the button center (320,180) was `SImage`, and the next click, at (400,300), hit `SNotificationBackground`. Slate's routing and `TopWindowAtPoint` both named that notification window, so the handler honestly delivered the click there. Its "delivered to window ''" looked like the fixture, which is also untitled. The `#3` stale-cursor theory was not the offscreen cause. The `#3` gate still stands for visible Linux hosts, where the drive route gate (commit 61c243f5) measured the stale cache. So the `#3` fix is partly right: the always-true handled bool was a real false-success source, while the stale cache is unproven offscreen. Test fix in `TestEditorHandlers.cpp`: `MouseClickHonestSuccess` makes its fixture `.IsTopmostWindow(true)` and asserts that `FDriveInput::TopWindowAtPoint(ClickPoint)` is the fixture before clicking. The comment is rewritten to the measured cause. New deterministic counterfactual `PinWright.editor.simulate_input.MouseClickOnNoWindowFails` clicks at (-100000,-100000) and requires `INPUT_FAILED`. The pre-`#3` handler answered success there because `ProcessMouseButtonDownEvent` always returns true. Debug logging removed. Verified by builds a1b960c3 (succeeded) and three offscreen runs with filter `PinWright.editor.simulate_input+PinWright.drive.click_occlusion`: c8a62ad3, c27f19d7 and 2777dc15, each 10/10 succeeded. `MouseClickHonestSuccess` and `MouseClickOnNoWindowFails` passed with no skip in all three. The only skip marker is `drive.click_occlusion.UncoveredTargetIsClicked`, from the same notification window over its fixture, tracked in `B-offscreen-notification-covers-fixtures`, which replaces the `B-offscreen-stale-cursor-window` link in `#3`.
