---
id: F-ui-common-activatable-stack
title: "`ui.activatable_*` — runtime push/pop and inspection for `UCommonActivatableWidgetStack`"
status: DONE
severity: Medium
category: feature
tags: [ui, common-ui, activatable, runtime, stack]
---

# `ui.activatable_*` — runtime push/pop and inspection for `UCommonActivatableWidgetStack`

CommonUI is a stock UE plugin used widely (Lyra-derived projects, many AAA samples) and ships with a layered runtime model: `UCommonActivatableWidgetStack` / `UCommonActivatableWidgetContainerBase` host `UCommonActivatableWidget` instances pushed via `AddWidget<T>()` and popped via `RemoveWidget()`. There is no typed MCP handler for this push/pop lifecycle today.

## Current state

- `ui.create_hud` adds a single widget to the player viewport. It does not target a `UCommonActivatableWidgetStack` inside an existing widget tree and does not push through CommonUI's transition / focus / input-suspend machinery.
- `ui.set_widget_visibility` can hide a widget but bypasses CommonUI activation semantics (`OnActivated` / `OnDeactivated`, input mode, focus restore).
- `widget.describe` (asset-side) shows the stack widget in the tree but does not report which `ActiveWidget` is currently on top at runtime.
- `python.execute` can reach `UCommonActivatableWidgetStack::AddWidget` only via reflection on the templated API; ergonomics are poor and unsafe across UE 5.4–5.7.

## Scope of this ticket

**In scope — runtime instance ops on an already-instantiated stack widget (parallel to existing `ui.*`):**

- `ui.activatable_push` — given a host widget instance name + stack-widget name + activatable widget-blueprint class path, instantiate the activatable and push it onto the stack. Returns the new instance's name.
- `ui.activatable_pop` — pop the top activatable off the named stack. Optional `instanceName` to pop a specific entry.
- `ui.list_stack_widgets` — list all activatables currently on the named stack, top → bottom, with class names and instance names.
- `ui.get_active_widget` — return the currently active (top) widget's instance name + class for the named stack.

Requires an active PIE session like the rest of `ui.*`.

**Explicitly out of scope (do NOT bundle):**

- *Authoring an activatable widget blueprint* — `widget.create_widget_blueprint` already accepts `parentClass`, so `parentClass: "CommonActivatableWidget"` covers this. No new sugar handler needed.
- *Setting `CommonButtonBase` style refs* — already covered by `widget.set` / `property.set` against the style UPROPERTY. Filing a typed wrapper would duplicate existing surface.
- *Carousel mutation* — separate widget class with its own semantics; if needed, file as its own ticket.

## Why a separate namespace was rejected

The original proposal suggested `common_ui.*` or `widget.common.*`. The push/pop/list/get-active operations are runtime ops on live widget instances, which is exactly the existing `ui.*` charter ("instance API, gone after PIE stops"). Putting them under `ui.activatable_*` keeps the asset/instance split clean and avoids a one-purpose namespace.

## Why it matters

CommonUI's activation model is the standard UE menu/HUD pattern post-5.1. Agents driving PIE sessions for UI verification (screenshot diffing, focus-state checks, input-mode reproduction) currently have no clean way to drive the stack — they fall back to triggering BP-level "push menu" events that may not exist on the test asset. A typed handler unblocks UI test automation against any CommonUI-based project.

## Implementation notes

- All four methods only need `#include "CommonActivatableWidget.h"` + `CommonActivatableWidgetContainerBase.h"` (CommonUI runtime module is already a project dep wherever this matters).
- `AddWidget` is a template; call the runtime non-template overload `AddWidgetInstance(UCommonActivatableWidget&)` after constructing via `CreateWidget<UCommonActivatableWidget>(OwningPlayer, Class)`.
- Tests: pin against a minimal fixture widget blueprint containing one `CommonActivatableWidgetStack` plus two trivial activatable BPs; push → list → pop → list and assert ordering.

## History
- `#1-filed-runtime-only-scope` `OPEN` reporter — Filed in response to a broader CommonUI proposal that bundled authoring + style + runtime. Authoring half is already covered by existing `widget.*` handlers (`parentClass` argument, `widget.set` on style UPROPERTYs). Filing only the runtime-stack half, which is genuinely missing and naturally fits `ui.*`. Carousel deliberately deferred to a separate ticket if demand appears.
- `#2-implemented-activatable-stack-rpcs` `IN-REVIEW` developer — Added `ui.activatable_push`, `ui.activatable_pop`, `ui.list_stack_widgets`, `ui.get_active_widget` in `Handlers/UI/UiActivatableStackHandler.cpp` over `UCommonActivatableWidgetContainerBase::{AddWidgetInstance, RemoveWidget, GetActiveWidget, GetWidgetList}`. No Build.cs change — CommonUI is transitive via CommonGame. Regression test `FUiActivatablePushBadClassTest` in `Tests/UI/TestUiActivatableStackHandlers.cpp` asserts CLASS_NOT_FOUND on an unloadable widget class; counterfactual: removing the class-resolution check makes the handler silently succeed.
- `#3-verify-fix` `DONE` tester — Verified: schema discovery (`?` suffix) on all four methods returned the documented `host`/`stack`/`widgetClass`/`instanceName` params; `ui.activatable_push` with bogus class `/Game/DoesNotExist/W_Bogus.W_Bogus_C` returned `error.code = "CLASS_NOT_FOUND"` matching the regression-test contract.
