---
id: B-simulate-input-cef-click-noop
title: "editor.simulate_input mouse_click injects a malformed Slate event (nullptr window, no effecting button) and hardcodes success — clicks silently miss the widget yet report {success:true}"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, simulate_input, mouse_click, slate-injection, effecting-button, silent-false-success, cef, swebbrowser]
encounters: 1
lastSeen: 2026-07-03T20:16:05+03:00
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
