---
id: F-drive-web-action-parity
title: "drive.click/scroll/drag/hover/key on surface=web are synthetic-DOM actions without Slate-surface guarantees (occlusion refusal, settle), scroll/drag/hover/key have no live tests, and the wiki still says those four return SURFACE_NOT_SUPPORTED"
status: IN-REVIEW
severity: High
category: feature
tags: [drive, web, cef, webbrowser, drive.click, drive.scroll, drive.drag, drive.hover, drive.key, occlusion, settle, wiki, parity]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Full web-surface parity for drive.click, scroll, drag, hover and key

Plugin `adb239fd`, UE 5.8. `surface=web` drives an embedded UMG `UWebBrowser` (CEF) during PIE.
No competitor drives embedded web UI at all, so this surface is a differentiator and its gaps are
visible on the public comparison page.

## Current state (read from source)

- `drive.scroll/hover/key/drag` do **not** return `SURFACE_NOT_SUPPORTED`. Since 0.8.0 they branch
  to `FDriveWebHandlers::ScrollWeb/HoverWeb/KeyWeb/DragWeb` (`DriveActionHandlers.cpp:185,222,283,336`,
  `DriveWebHandlers.cpp:513-673`), which inject JS built by `FDriveWebBridge::Build{Scroll,Hover,Key,Drag}ElementJs`
  (`DriveWebBridge.cpp` ~838-936).
- All four dispatch **synthetic, untrusted** DOM events (`isTrusted:false`):
  - scroll: `WheelEvent` plus a `scrollBy` fallback on the element and nearest scrollable ancestor.
  - hover: `pointerover/mouseover/mouseenter/mousemove`; CSS `:hover` never applies.
  - key: `keydown/keypress/keyup` on the focused element; no default actions (no text insertion,
    no Tab focus move, no Enter activation).
  - drag: pointer/mouse down-move-up between two handles; no HTML5 `dragstart/dragover/drop` with
    `DataTransfer`, no intermediate moves, no `duration_ms`, no visibility check on either end.
- `drive.click` on web (`ClickWeb`, `DriveWebHandlers.cpp:450`) calls `el.click()` (`BuildClickElementJs`,
  `DriveWebBridge.cpp:798`): a synthetic click with the same missing occlusion refusal and one-shot settle.
- No occlusion refusal: the web path never hit-tests (`elementFromPoint`), so a covered target is
  "acted on" with `ok:true`. The Slate path refuses with `TARGET_OCCLUDED` (`DriveActionCommon.cpp:423`).
- Settle is one re-query after injection (`RunWebActionWithSettle`, `DriveWebHandlers.cpp:138`), not the
  change-then-stable loop with `stable_ticks` / `settle_budget_ms` / `quiet_budget_ms`.
- Tests: only JS-snippet string tests (`PinWright.drive.web.{Scroll,Hover,Key,Drag}JsSnippet`) and
  no-browser error tests (`PinWright.drive.webint.*NoBrowser*`, `DragMissingToHandle`). No live test
  proves any of the four changes a real page.
- Wiki `docs/wiki-src/drive.md` "Web limitations" (line 56-58) says the four verbs have no web mapping
  and return `SURFACE_NOT_SUPPORTED`, and calls web "v1 and partial". That is stale on the error claim
  and correct on "partial".

## Ask

Parity with the Slate surface for `drive.click` and the four verbs on `surface=web`:
- Real input where the page must see it as user input (trusted events, CSS `:hover`, key default
  actions, HTML5 drag-and-drop), e.g. via CEF-level mouse/key injection at the element's viewport point
  instead of `dispatchEvent`.
- Occlusion refusal with `TARGET_OCCLUDED` via a hit-test at the injection point (click included).
- The same settle model and before/after `diff` semantics as Slate where they apply.
- Genuinely unsupported cases keep their current typed errors (`TARGET_NOT_FOUND`, `TARGET_CHANGED`,
  `INVALID_ARGUMENT` for web drag without `to_handle`, `WEB_BROWSER_NOT_FOUND`).

**Comparison page:** the cell "Drive embedded web UI (WebBrowser/CEF) during play" grades PinWright
"yes". It should read "partial" until this ticket is `DONE`.

## Acceptance

- Live tests in the `PinWright.drive.web*` group, on a web fixture page, one per verb (scroll, drag,
  hover, key), each asserting the page-side effect and an occluded-target refusal.
- A live occluded-target refusal test for web `drive.click`.
- `docs/wiki-src/drive.md:56-58` "Web limitations" is stale (claims the four verbs return
  `SURFACE_NOT_SUPPORTED`) and must be corrected as part of this ticket, to the shipped behaviour.
- Typed errors above unchanged for unsupported cases.
- Verified on UE 5.8, Windows only.

## History
- `#1-filed-web-parity-gap` `OPEN` reporter - Filed at the user's request as High. The brief assumed the four verbs return `SURFACE_NOT_SUPPORTED` (per the wiki); source shows synthetic-event implementations with no occlusion check, one-shot settle and no live tests, so the ticket targets real parity plus the stale wiki.
- `#2-scope-adds-web-click` `OPEN` reporter - Scope widened to web `drive.click`: it uses `el.click()` with no occlusion hit-test and the same one-shot settle, so it needs the same Slate-surface guarantees. Made the stale `drive.md:56-58` wiki fix an explicit acceptance item.
- `#3-real-input-occlusion-settle` `IN-REVIEW` developer - Web `drive.click`/`hover`/`scroll`/`drag`/`key` now deliver real input. A new in-page locate step (`FDriveWebBridge::LocateElement` / `BuildLocateElementJs`) resolves the handle, refuses hidden/disabled as `TARGET_CHANGED`, scrolls an off-viewport center into view, hit-tests the center with `document.elementFromPoint` and refuses a covered target with `TARGET_OCCLUDED` (`occluding_element`, CSS `x`/`y`) before anything is injected. The CSS point is mapped to the desktop (`CssToLocal` over the browser widget's cached geometry) and injected through `FDriveInput` (`ClickAt`/`HoverAt`/`ScrollAt`/`DragFromTo`, the game surface's Slate path), behind a 0.3 s gate that requires Slate's hit-test at that point to reach the browser widget, so another window or a widget drawn over the browser is also `TARGET_OCCLUDED` (`occluding_window`/`occluding_widget`). Keys: the browser takes Slate keyboard focus, the optional handle is DOM-focused (no click), then key down + the platform character (`MapDomKey`: DOM names or FKey names, Enter `\r`, Backspace `\b`, printable keys) + key up; unknown names are `INVALID_KEY`. Settle is now `FDriveSettleDriver` (stable_ticks / quiet / settle budgets / wait_for + widget_absent baseline guard) stepped once per completed DOM query, and responses use the game shape `{outcome, changed, settled, condition_met, elapsed_ms, ticks, input_path, diff, observation?, journal?}` (drive.type too, `input_path:"dom"`; its injection is still the DOM value write). Web `delta` now matches Slate (notches, positive up); `button` and `duration_ms` are honoured; `os_input` on web is refused with `INVALID_ARGUMENT`. The synthetic Build{Click,Scroll,Hover,Key,Drag}ElementJs builders and their bridge methods are removed. HTML5 drag-and-drop (`dragstart`/`drop`) is NOT delivered and cannot be from the plugin: the engine's CEF host never implements `CefRenderHandler::StartDragging`, so Chromium aborts native drags; documented as a limitation, pointer/mouse drags work. Wiki: `docs/wiki-src/drive.md` "Web limitations" replaced by "Web surface", per-verb "Not supported on web" lines corrected. Files: `Source/PinWright/Private/Handlers/Drive/DriveWebBridge.{h,cpp}`, `DriveWebHandlers.{h,cpp}`, `Tests/Drive/TestDriveWebBridge.cpp`, new `Tests/Drive/TestDriveWebLive.cpp`, `docs/wiki-src/drive.md`, `CHANGELOG.md`. Tests: live `PinWright.drive.weblive.{ClickIsTrustedAndRefusesOccluded,HoverAppliesCssHoverAndRefusesOccluded,ScrollWheelsListAndRefusesOccluded,KeyTypesIntoInputAndRefusesOccluded,DragDropsOnTargetAndRefusesOccluded}` (own UWebBrowser fixture in a Slate window loading a `data:` page; skip with the marker when CEF/renderer is unavailable or the page never becomes drivable); pure `PinWright.drive.web.{LocateJsSnippet,ParseLocateResult,CssToLocal,MapDomKey}` replace the removed `{Click,Scroll,Hover,Key,Drag}JsSnippet`. Verified on Linux UE 5.8 by compile only so far; live run pending. The comparison-page cell is not in the plugin repo and was not touched.
- `#4-offscreen-routing-fix` `IN-REVIEW` developer - Round-1 suite: `core.error_codes.RegistryAdoptingFilesUseConstantsOnly` failed on hand-spelled codes in `DriveWebBridge.cpp` (now `ErrorCodes::ERR_TIMEOUT` / `ERR_MALFORMED_JSON` / `ERR_WEB_BROWSER_NOT_FOUND`; the unregistered `NO_BROWSER` and the redundant `Code = "OK"` are gone). Four of the five `drive.weblive.*` tests skipped as `fixture-window-stacked-under-host-window`; `KeyTypesIntoInputAndRefusesOccluded` measured and passed. Cause not measured in this run; two candidates, both now handled: (a) the host window sits above the fixture in Slate's window order on the small offscreen display (what `B-offscreen-notification-covers-fixtures` measured for the notification window), and (b) under `-RenderOffScreen` Linux uses SDL's `dummy` video driver, where `FLinuxCursor::SetPosition` warps inside whichever window already holds SDL mouse focus, so the platform window-under-cursor that `LocateWindowUnderMouse` consults first can stay pinned to a host window and `ProcessMouse*`-based injection (`FDriveInput`) lands there. Web pointer input no longer goes through `FDriveInput`/the platform: the path is taken from Slate's own window order (`FDriveInput::TopWindowAtPoint`) and that window's hit-test grid, must contain the browser widget (else `TARGET_OCCLUDED` with `occluding_window`/`occluding_widget`, nothing pressed), and the events go through `FSlateApplication::RoutePointerMove/Down/Up/RouteMouseWheelOrGestureEvent` along it (platform cursor moved along); the 0.3 s warp wait is gone. The live fixture window is now topmost and placed clear of every visible window (`GetAllVisibleWindowsOrdered`), so the skip remains only for a window opened over it afterwards. Same root cause very likely explains the Slate-surface `drive.click_occlusion.UncoveredTargetIsClicked` skip in the same run (not changed here).
