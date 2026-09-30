---
id: F-drive-web-action-parity
title: "drive.click/scroll/drag/hover/key on surface=web are synthetic-DOM actions without Slate-surface guarantees (occlusion refusal, settle), scroll/drag/hover/key have no live tests, and the wiki still says those four return SURFACE_NOT_SUPPORTED"
status: OPEN
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
