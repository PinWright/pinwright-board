---
id: B-drive-web-omits-custom-interactables
title: "drive web observe omits custom ARIA/tabindex controls (role=listbox etc.) from interactables — no clickable handle"
status: IN-REVIEW
severity: High
category: bug
tags: [drive, web, cef, observe, interactable-classification, aria, tabindex, custom-dropdown]
encounters: 1
lastSeen: 2026-07-03T17:15:35.0000000Z
---

# drive web observe omits custom ARIA/tabindex controls from interactables

## What's wrong
On `surface:web`, `drive.observe` classifies interactability with a fixed CSS
selector that only covers native controls plus `role=button`/`onclick`. Any
custom control built from a focusable div with an interactive ARIA role — the
project's dropdown, `<div class="pdd" tabindex="0" role="listbox">` — matches
neither the interactable selector nor the text selector, so it is emitted with
**no handle at all** (only its child value `<span>` shows up, as
`interactable:false`). `drive.click` therefore has no handle to target, and the
project's CEF physics/HUD panels use `.pdd` dropdowns pervasively. The
DOM-injection click path itself works fine for the handles that DO get emitted
(header divs with `role="button"` click and re-render correctly) — the defect is
purely that the classifier under-covers standard interactive controls.

## Guilty source (ground truth, read verbatim)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveWebBridge.cpp:769`
  (`BuildQueryElementsJs`): the interactable selector is
  `var IS='a,button,input,select,textarea,[role=button],[onclick]';` — no
  `[tabindex]`, no `[role=listbox|combobox|option|menuitem]`.
- Line 776 `document.querySelectorAll(IS).forEach(el=>add(el,true))` marks IS
  matches `interactable:true`; line 777 marks the text selector
  `TS='h1..h6,p,span,label,li,td,th'` (770) as `interactable:false`. A `.pdd`
  `div` matches neither, so `add()` is never called on it — it gets no handle.
  `ParseQueryResult` (962-1032) just copies the JS-side `interactable` flag; no
  C++ path can recover the omission.

## Session evidence (replayable)
- Live PIE drone-edit CEF panel, Constructor expanded:
  `drive.observe {surface:web, interactables_only:true}` returned only 4
  handles, all section headers (`БЭКЕНД`, `КОНСТРУКТОР`,
  `РАСШИРЕННЫЕ НАСТРОЙКИ ▾`, `НАСТРОЙКИ ОЩУЩЕНИЙ`). None of the 5 constructor
  `.pdd` dropdowns nor the 6 feel-preset `.pdd` dropdowns appeared.
- Full `drive.observe {surface:web}` listed the dropdowns' value spans as
  `interactable:false` (e.g. `pw-10` "4S 1550mAh 100C"), never the parent `.pdd`
  div as a handle.
- Contrast: `drive.click {surface:web, handle:"pw-1"}` on an exposed header div
  worked (expanded the section, DOM re-rendered) — DOM-injection click is fine.

## Related
Two siblings from the same investigation (file separately): a Slate-click-over-CEF
no-op (`B-simulate-input-cef-noop` — injected Slate mouse doesn't fire DOM
handlers) and a duplicate-`pw-0` handle collision (`B-drive-web-handle-collision`).
Combined with the Slate-noop finding, the custom dropdown is unclickable by any
route. This ticket is scoped to interactable **detection** only.

**Workaround:** None reliable. `interactables_only:true` (the standard discovery
mode) never surfaces the `.pdd` control. A full observe does stamp a `data-pw-id`
on the child value span, so `drive.click` on that child handle is a fragile,
untested possibility (relies on the click bubbling to the parent `.pdd` handler).

**Fix:** Extend the `IS` selector in `BuildQueryElementsJs`
(`DriveWebBridge.cpp:769`) to also match `[tabindex]` (focusable) and interactive
ARIA roles — `[role=listbox]`, `[role=combobox]`, `[role=option]`,
`[role=menuitem]`, `[role=tab]`, `[role=switch]`, `[role=checkbox]`,
`[role=radio]` — so custom controls get an `interactable:true` handle.

severity rationale: impact=hard blocker (the standard `interactables_only`
discovery path returns zero usable handles for the dominant control type on the
project's web HUD; no reliable workaround) × reach=web driving is a sub-path but
custom `.pdd` dropdowns are pervasive on every physics/HUD panel -> High

## History
- `#1-initial-repro` `OPEN` reporter — Filed: on `surface:web`, `drive.observe`
  classifies interactability with a fixed selector
  `IS='a,button,input,select,textarea,[role=button],[onclick]'`
  (`DriveWebBridge.cpp:769`) that excludes `[tabindex]` and interactive ARIA
  roles. The project's `<div class="pdd" tabindex="0" role="listbox">` dropdown
  matches neither IS nor the text selector, so it is emitted with no handle and
  `drive.click` cannot target it. Live repro: constructor panel
  `observe {interactables_only:true}` returned only 4 section-header handles;
  none of the 11 `.pdd` dropdowns appeared, while a full observe listed only
  their child value spans as `interactable:false`. DOM-injection click confirmed
  working on exposed header handles, so the gap is detection-only. Fix: add
  `[tabindex]` + ARIA interactive roles to the IS selector.
- `#2-implement-selector-fix` `IN-REVIEW` developer — Root-cause fix implemented.
  Extended the interactable selector `IS` in `BuildQueryElementsJs`
  (`DriveWebBridge.cpp:768` — ticket cited :769, off by one; content exact) from the
  native-only set to also match focusable elements and interactive ARIA widget roles:
  `[tabindex]:not([tabindex="-1"])` (tightened past the ticket's bare `[tabindex]` so
  programmatic-only `tabindex="-1"` focus targets are not over-emitted) plus
  `[role=link|checkbox|radio|switch|tab|menuitem|menuitemcheckbox|menuitemradio|option|combobox|listbox|slider|spinbutton|textbox|searchbox|treeitem]`.
  A `<div class="pdd" tabindex="0" role="listbox">` now matches `IS`, so `add(el,true)`
  stamps it a `data-pw-id` handle with `interactable:true` and `drive.click` can target it.
  Detection-only change; no C++ parser edit needed (`ParseQueryResult` copies the JS flag
  verbatim). Files: `Source/PinWright/Private/Handlers/Drive/DriveWebBridge.cpp` (selector),
  `Source/PinWright/Private/Tests/Drive/TestDriveWebBridge.cpp` (added regression test
  `PinWright.drive.web.QueryJsInteractableSelector`, which asserts the produced query JS
  covers `[tabindex]` + the ARIA roles and fails if the selector is reverted). Plugin builds
  clean. Not a duplicate of the same-function `B-drive-web-handle-collision` (disjoint
  `hid()` minter) nor the different-file `B-simulate-input-cef-click-noop`.
