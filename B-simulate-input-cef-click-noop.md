---
id: B-simulate-input-cef-click-noop
title: "editor.simulate_input mouse_click over a CEF SWebBrowserView reports success but doesn't trigger DOM click handlers"
status: OPEN
severity: High
category: bug
tags: [editor, simulate_input, mouse_click, cef, swebbrowser, webui, silent-false-success, slate-injection]
encounters: 1
lastSeen: 2026-07-03T20:16:05+03:00
---

# editor.simulate_input mouse_click over a CEF browser is a phantom-success no-op

`editor.simulate_input {type:"mouse_click", x, y}` at coordinates over an embedded
CEF browser (`SWebBrowserView`) returns `{success:true}` but the web page's DOM
click handler never fires — the injected Slate pointer event is not forwarded into
CEF's DOM. There is no other coordinate-based way to click CEF content; `drive.click`
(DOM injection) works but only for elements the web observer exposes as handles, so a
custom `.pdd` dropdown (no handle) is unclickable by any automation path.

The response is a **silent lie**: the handler hardcodes `bSuccess = true`
(`EditorCommandHandler.cpp:773`) regardless of whether any widget consumed the event —
the bool returns of `ProcessMouseButtonDownEvent`/`ProcessMouseButtonUpEvent` are
discarded — so a caller who clicked nothing is told the click landed.

## Guilty source (Slate-only injection, no CEF forwarding)
`Handlers/Editor/EditorCommandHandler.cpp` — `editor.simulate_input` registered at
line 695; the `mouse_click` branch (lines 745-774) builds two `FPointerEvent`s
(762-771) and routes them via `SlateApp.ProcessMouseButtonDownEvent(nullptr, MouseDownEvent)`
(765) + `ProcessMouseButtonUpEvent(MouseUpEvent)` (771). It never touches
`SWebBrowserView` / `IWebBrowserWindow` / any CEF API — it relies entirely on Slate's
own hit-test/route (with a `nullptr` platform window), which does not reach the CEF
browser's DOM. `bSuccess = true` at line 773 is unconditional.

## Session evidence (replayable)
Live PIE, CEF physics panel. `PhysicsWebBrowser`/`SWebBrowserView` rect (from
`drive.observe(game)`): x=1761 y=488 w=673 h=523. Target `.pdd` dropdown ("4S 1550mAh
100C") DOM geometry (from `drive.observe(web)`): x=2268 y=761 w=113 -> click center
~(2324,772), inside the browser rect.
1. `editor.simulate_input {type:"mouse_move", x:2324, y:772}` -> `{success:true}`.
2. `editor.simulate_input {type:"mouse_click", x:2324, y:772, button:"left"}`
   -> `{success:true, message:"Mouse click at (2324.000000, 772.000000)"}`.
3. Screenshot immediately after: panel UNCHANGED, dropdown did not open — DOM click
   handler never fired. Contrast: `drive.click` (DOM injection) reliably actuated
   exposed handles in the same panel.

**Workaround:** For CEF elements the web observer exposes as handles, use `drive.click`
(DOM injection). Custom controls with no handle have no working path.

**Fix:** Forward synthesized mouse events into the focused `SWebBrowserView`/CEF host
(so DOM handlers fire), or add a coordinate/handle click path on `surface:web` in
`drive`. At minimum, stop hardcoding `bSuccess=true` (honor the `Process*` return) and
document that `simulate_input` does not reach CEF DOM.

Related (cross-ref): `F-drive-web-omits-custom-interactables` (the handle gap that makes
`drive.click` unusable here), `B-drive-web-handle-collision`.

severity rationale: impact=silent-false-success — `bSuccess=true` is hardcoded, so a
click that actuated nothing returns affirmative success the caller builds on — AND a
hard blocker with no workaround for handle-less custom CEF controls (`drive.click`
can't reach them). Reach=CEF panels are a real, shipped surface in this project (the PDS
physics WebUI), not a one-off. Both point to High.

## History
- `#1-cef-click-noop` `OPEN` reporter — Filed: `editor.simulate_input {mouse_click}` at coords over a live CEF `SWebBrowserView` (PDS physics panel) returned `{success:true, message:"Mouse click at (2324,772)"}` but the post-click screenshot showed the panel unchanged — the DOM click handler never fired. Confirmed against source: `EditorCommandHandler.cpp:745-774` builds `FPointerEvent`s and routes them through `SlateApp.ProcessMouseButtonDownEvent(nullptr,...)` / `ProcessMouseButtonUpEvent(...)` (765,771) with zero CEF/`SWebBrowserView`/`IWebBrowserWindow` forwarding, and hardcodes `bSuccess=true` at line 773 (discarding the `Process*` handled-bool) — a silent false-success. No coordinate-based path reaches CEF DOM; `drive.click` (DOM injection) works but only for observer-exposed handles, so handle-less custom `.pdd` controls are unclickable by any means. No existing board file mentions `simulate_input` or CEF/SWebBrowser click (ripgrep clean). Note: named sibling tickets `F-drive-web-omits-custom-interactables` / `B-drive-web-handle-collision` are not yet on the board (likely filed concurrently).
