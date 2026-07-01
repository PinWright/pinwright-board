---
id: E-activatable-host-runtime-instance-name-undocumented
title: "`ui.activatable_*` `host` must be the runtime instance name from `ui.create_hud`, not the asset/class name — undocumented, errors `HOST_NOT_FOUND`"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [ui, activatable_push, list_stack_widgets, common-ui, host-resolution, docs]
---

# `ui.activatable_*` `host` silently requires the live instance name, not the widget asset/class name

The four `ui.activatable_*` / `ui.list_stack_widgets` methods (`activatable_push`,
`activatable_pop`, `list_stack_widgets`, `get_active_widget`) all take a `host`
param that resolves the HUD widget the stack lives inside. The caller naturally
passes the name they *authored* — the widget blueprint asset/class name
(`WBP_PauseHost`) — and gets:

```
[HOST_NOT_FOUND] Host widget 'WBP_PauseHost' not found in any live world
```

The call only succeeds once `host` is the **runtime instance name** that
`ui.create_hud` returned a moment earlier (`WBP_PauseHost_C_0`). Nothing tells the
caller that `host` is keyed on the live-instance name (auto-suffixed `_C_<n>`)
rather than the asset/class name they just instantiated — the `host` param is
described only as a "host widget instance name" with no example, no statement that
it is the `ui.create_hud` return value, and no `?`-schema hint that distinguishes
it from a class path. The error reads like "no such widget exists" rather than
"you gave me the asset name where I wanted the live-instance name." The caller is
forced into a trial-and-error retry to discover the right form.

## Why this is the PROCESS angle distinct from the judge's `_C` ticket

This task's `ui.activatable_push` took **three** attempts, not one — two *separate*
discovery gaps stacked back-to-back:

1. `widgetClass:"…WBP_PauseMenu.WBP_PauseMenu"` (bare) → `CLASS_NOT_FOUND` — the
   `widgetClass`/`_C`-suffix gap, already filed as
   `E-activatable-push-requires-c-suffix` (the per-finding judge's ticket).
2. `widgetClass:"…WBP_PauseMenu_C"` + `host:"WBP_PauseHost"` (asset name) →
   **`HOST_NOT_FOUND`** — *this* ticket: the `host` value is wrong, not the class.
3. `widgetClass:"…WBP_PauseMenu_C"` + `host:"WBP_PauseHost_C_0"` (live instance) →
   success.

The judge's ticket is scoped to `widgetClass`/`_C` resolution (its title, fix,
tags, and proposed code change are all about routing `widgetClass` through
`ResolveUClass`); it only mentions the host-naming gap in passing as something the
wiki "could note." That host gap is a *separate* retry with a *different* error
code on a *different* param, and it applies to **all four** `ui.activatable_*`
methods (every one takes `host`), not just `push`. Filing it as its own docs ticket
so the host-naming discovery gap isn't lost when the `_C` ticket is fixed.

## Evidence (this audit — friction note + call-log)

Friction note, verbatim:

> "host must be the live instance name `WBP_PauseHost_C_0` returned by
> `ui.create_hud` rather than the asset/class name (HOST_NOT_FOUND)."

Call-log: `ui.create_hud {widgetPath: WBP_PauseHost} → {widgetName:"WBP_PauseHost_C_0"}`,
then `ui.activatable_push {host:"WBP_PauseHost", …}` → `is_error` `HOST_NOT_FOUND`,
then `ui.activatable_push {host:"WBP_PauseHost_C_0", …}` → ok. The corrected value
is exactly what `ui.create_hud` returned, so the whole detour is a discoverability
gap: the wiki never links "the `widgetName` `ui.create_hud` returns" to "the `host`
the `ui.activatable_*` methods want."

## What it should do

- Wiki overlay to improve: `docs/wiki-src/ui.md`. The page now carries a
  `### ui.activatable_push` section (added by the sibling
  `E-activatable-push-requires-c-suffix`, `ui.md:23-27`), but that section covers
  only `widgetClass`/`_C` resolution and says nothing about how `host` is keyed;
  no overlay section exists yet for `activatable_pop` / `list_stack_widgets` /
  `get_active_widget`. **Extend** the existing `### ui.activatable_push` H3 (and
  **add** matching H3 sections for the other three methods) stating that `host` is
  the **runtime instance name returned by `ui.create_hud`'s `widgetName`** (the
  auto-suffixed `WBP_…_C_<n>` live-instance name), *not* the widget blueprint
  asset/class name, and give one worked example chaining `ui.create_hud` →
  `ui.activatable_push` using the returned name. The `ui.create_hud` prelude note
  ("the returned widget reference can be addressed by name in subsequent
  `ui.set_widget_*` calls") should also name `ui.activatable_*` alongside
  `ui.set_widget_*`.
- Optionally, the `HOST_NOT_FOUND` error text should hint that `host` expects the
  live-instance name (`…_C_<n>`) returned by `ui.create_hud`, when the value given
  matches a known widget asset/class rather than a live instance.

**Workaround:** Pass the `widgetName` that `ui.create_hud` returned (e.g.
`WBP_PauseHost_C_0`) as `host`, not the asset/class name.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-auditor PROCESS finding from a layered pause-menu push/pop/list task (seed `ui.list_stack_widgets`). `ui.activatable_push` needed three attempts: bare `widgetClass` → `CLASS_NOT_FOUND` (filed separately as `E-activatable-push-requires-c-suffix`), then `_C` class + `host:"WBP_PauseHost"` (asset name) → `HOST_NOT_FOUND`, then `host:"WBP_PauseHost_C_0"` (the live-instance `widgetName` returned by `ui.create_hud`) → success. The host-naming gap is a *separate* retry/error code on a *different* param than the judge's `_C` ticket, and applies to all four `ui.activatable_*` methods. Both gaps were recoverable from the error text but neither is documented. Wiki overlay to improve: `docs/wiki-src/ui.md` (the `host`↔`ui.create_hud` `widgetName` linkage is undocumented). NOTE (reword): the sibling `E-activatable-push-requires-c-suffix` has since added a `### ui.activatable_push` section at `ui.md:23-27`, but it covers only `widgetClass`/`_C` — the host-naming gap remains unaddressed; the fix now extends that section rather than creating one from scratch.
- `#2-reword-and-document-host` `IN-REVIEW` developer — Reworded the stale premise (the body + History no longer claim "no `ui.activatable_*` section exists"; the `### ui.activatable_push` section already exists at `ui.md:23-27`, scoped to `widgetClass`/`_C` only — the **host**-naming gap was still unaddressed). Fix rescoped from "create a section" to "extend the existing `### ui.activatable_push` section + add the three missing method sections." Documented that `host` is matched against the **live runtime instance name** (`FindStackInPie` compares `It->GetName()` at `UiActivatableStackHandler.cpp:26`) — the auto-suffixed `WBP_…_C_<n>` `widgetName` that `ui.create_hud` returns (`UiHandler.cpp:337`), NOT the asset/class name (which yields `HOST_NOT_FOUND`). Files: `docs/wiki-src/ui.md` — (a) extended the `### ui.activatable_push` H3 with the host-naming note + a worked `ui.create_hud` → `ui.activatable_push` chain example; (b) added `### ui.activatable_pop`, `### ui.list_stack_widgets`, `### ui.get_active_widget` H3 sections each carrying the same host note (none existed before); (c) extended the `ui.create_hud` prelude note to name `ui.activatable_*` alongside `ui.set_widget_*` and state that the returned `widgetName` is the value `host` wants. Regression test: `Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp::FWikiHandlerActivatableHostInstanceNameDocumentedTest` (`PinWright.infra.wiki_handler.MethodPage.ActivatableHostInstanceNameDocumented`) renders all four method pages via `WikiHandler::RenderPage` and asserts each carries the overlay-exclusive markers `ui.create_hud`, `widgetName`, `_C_<n>`, and `HOST_NOT_FOUND` — all absent from the handler registrations/param descriptions, so reverting any of the four H3 sections drops the markers and fails the test. Did not compile/run (later phase). Distinct from the sibling `E-activatable-push-requires-c-suffix` (IN-REVIEW): different param (`host` vs `widgetClass`), different error code (`HOST_NOT_FOUND` vs `CLASS_NOT_FOUND`), all four methods vs push only.
