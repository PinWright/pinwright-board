---
id: E-activatable-push-requires-c-suffix
title: "`ui.activatable_push` rejects the bare Blueprint asset path on `widgetClass`, silently requiring the undocumented `_C` generated-class suffix"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [ui, activatable_push, common-ui, class-resolution, docs]
---

# `ui.activatable_push` rejects the bare asset path on `widgetClass`, requiring an undocumented `_C` suffix

`ui.activatable_push {host, stack, widgetClass:"/Game/ReproUI/WBP_ReproMenu.WBP_ReproMenu"}`
— the bare `<Path>.<Asset>` form that `widget.create_widget_blueprint` *returns*
as its `widgetPath`, and that the sibling `ui.create_hud` explicitly accepts —
fails with:

```
[CLASS_NOT_FOUND] Failed to load UCommonActivatableWidget class: /Game/ReproUI/WBP_ReproMenu.WBP_ReproMenu
```

The call only succeeds once the caller appends the generated-class suffix `_C`
(`...WBP_ReproMenu.WBP_ReproMenu_C`). Nothing in the `ui.activatable_push` wiki
section, the `widgetClass` param description ("Class path of the
UCommonActivatableWidget subclass to instantiate"), or the error text tells the
caller this — the error reads like the asset is missing rather than "you gave me
an asset path where I wanted a generated-class path." The caller is forced into a
trial-and-error retry to discover the required form.

## Why this is awkward (resolver-consistency gap — quotable contradiction)

The `ui.create_hud` wiki page (`docs/wiki-src/ui.md`, `### ui.create_hud`) now
documents, verbatim:

> "`widgetPath` accepts the bare Blueprint asset path
> (`/Game/.../WBP_PlayerHUD.WBP_PlayerHUD`) — the generated-class `_C` suffix is
> resolved for you and is optional, **matching every other class-path slot in the
> API**. A short class name (`WBP_PlayerHUD`) also resolves."

`ui.activatable_push`'s `widgetClass` is a class-path slot in the same `ui.*`
namespace, yet it is **not** one of the "every other class-path slot" that
resolves the bare path — it concretely contradicts the documented API invariant.
The handler does a raw `LoadClass<UCommonActivatableWidget>(nullptr, *ClassPath)`
(`Source/PinWright/Private/Handlers/UI/UiActivatableStackHandler.cpp:91`), which
does not append `_C` and rejects the bare asset path before the host/stack lookup
ever runs.

This is the *same* defect class that `E-class-name-format-inconsistency` (DONE)
built `ClassUtils.cpp::ResolveUClass` to fix — that helper tries direct load,
`_C`-appended load, then `LoadObject<UBlueprint>` → `GeneratedClass`, so callers
never have to type `_C`. That sweep routed `inspect_class`,
`widget.create_widget_blueprint`, and `asset.search_assets` through `ResolveUClass`;
a follow-up (`E-create-hud-requires-c-suffix-class-path`, IN-REVIEW) routed
`ui.create_hud`. `ui.activatable_push` (added later in
`F-ui-common-activatable-stack`) was in **neither** sweep and still does the raw
`LoadClass`. Note the verification of `F-ui-common-activatable-stack` (#3) only
ever exercised the `_C` form (`/Game/DoesNotExist/W_Bogus.W_Bogus_C`), so the
bare-path gap was never noticed.

## Verbatim repro (this audit — live PIE)

1. `widget.create_widget_blueprint {name:"WBP_ReproHost", folder:"/Game/ReproUI", parentClass:"UserWidget"}` → ok, `widgetPath:"/Game/ReproUI/WBP_ReproHost.WBP_ReproHost"`
2. `widget.create_widget_blueprint {name:"WBP_ReproMenu", folder:"/Game/ReproUI", parentClass:"CommonActivatableWidget"}` → ok, `widgetPath:"/Game/ReproUI/WBP_ReproMenu.WBP_ReproMenu"`
3. `widget.add {widgetPath:"/Game/ReproUI/WBP_ReproHost.WBP_ReproHost", type:"CommonActivatableWidgetStack", name:"MenuStack", parentName:"RootCanvas"}` → ok
4. compile + save both; `editor.play {}`; `ui.create_hud {widgetPath:"/Game/ReproUI/WBP_ReproHost.WBP_ReproHost"}` → `{widgetName:"WBP_ReproHost_C_0"}`
5. `ui.activatable_push {host:"WBP_ReproHost_C_0", stack:"MenuStack", widgetClass:"/Game/ReproUI/WBP_ReproMenu.WBP_ReproMenu"}` (bare asset path)
   → `is_error` `[CLASS_NOT_FOUND] Failed to load UCommonActivatableWidget class: /Game/ReproUI/WBP_ReproMenu.WBP_ReproMenu`
6. `ui.activatable_push {host:"WBP_ReproHost_C_0", stack:"MenuStack", widgetClass:"/Game/ReproUI/WBP_ReproMenu.WBP_ReproMenu_C"}` (with `_C`)
   → success `{instanceName:"WBP_ReproMenu_C_0", className:"WBP_ReproMenu_C"}`

Identical bare path, only `_C` differs — same asset, so the load failure is purely
the missing `_C` append, not a missing asset.

## What it should do

- Route `ui.activatable_push`'s `widgetClass` through `ResolveUClass` (instead of
  the raw `LoadClass<UCommonActivatableWidget>(nullptr, *ClassPath)` at
  `UiActivatableStackHandler.cpp:91`) so the bare
  `/Game/<Path>/<Blueprint>.<Blueprint>` path (and short names) resolve to the
  generated class without `_C` — matching `ui.create_hud`, `widget.add`, and the
  resolver contract from `E-class-name-format-inconsistency`. Keep the existing
  `IsChildOf(UCommonActivatableWidget)` guard after resolution so the downstream
  `CreateWidget<UCommonActivatableWidget>` cast stays type-safe (`ResolveUClass`
  returns any `UClass*`).
- Until the resolver lands, the `ui.activatable_push` wiki overlay should state
  that `widgetClass` is a **generated-class** path requiring `_C`
  (`/Game/.../WBP_Menu.WBP_Menu_C`), and the `CLASS_NOT_FOUND` error should hint
  "append `_C` for a Blueprint widget" when a bare asset path was given.

**Workaround:** Append `_C` to the asset path passed as `widgetClass`.

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode audit of `ui.list_stack_widgets` (a layered pause-menu push/pop/list task). The seed and all four `ui.activatable_*` ops worked; the friction is the neighbor `ui.activatable_push`, whose `widgetClass` rejects the bare `/Game/ReproUI/WBP_ReproMenu.WBP_ReproMenu` path with `[CLASS_NOT_FOUND]` and only accepts the `_C` form. Replay-confirmed live in PIE (steps 5/6 above): identical path, only `_C` differs, same asset. Root cause: raw `LoadClass<UCommonActivatableWidget>(nullptr, *ClassPath)` at `UiActivatableStackHandler.cpp:91` bypasses `ResolveUClass`. Distinct from `E-create-hud-requires-c-suffix-class-path` (same defect class, different method — `ui.create_hud`, already IN-REVIEW) and from the umbrella `E-class-name-format-inconsistency` (DONE) sweep, which never covered `ui.activatable_push`. Quotable contradiction: the `ui.create_hud` wiki claims the `_C` suffix is optional "matching every other class-path slot in the API," which `ui.activatable_push`'s slot violates.
- `#2-route-through-resolveuclass` `IN-REVIEW` developer — Routed `ui.activatable_push`'s `widgetClass` through `ResolveUClass` instead of the raw `LoadClass<UCommonActivatableWidget>(nullptr, *ClassPath)`, so the bare `/Game/.../WBP.WBP` asset path (and short names) resolve to the generated class without the `_C` suffix — a one-region transplant of the IN-REVIEW `E-create-hud-requires-c-suffix-class-path` fix on the sibling `ui.create_hud`. Kept the existing `IsChildOf(UCommonActivatableWidget)` guard after resolution (ResolveUClass returns any `UClass*`) so the downstream `CreateWidget<UCommonActivatableWidget>` cast stays type-safe. Files: `Source/PinWright/Private/Handlers/UI/UiActivatableStackHandler.cpp` (added `#include "PinWrightHelpers.h"`; swapped the load at the former line 91), `Docs/wiki-src/ui.md` (new `### ui.activatable_push` section documenting that `widgetClass` accepts the bare path, matching the create_hud contract). Regression test: `Source/PinWright/Private/Tests/UI/TestUiActivatableStackHandlers.cpp::FUiActivatablePushBareClassPathResolvesTest` (`PinWright.ui.activatable.PushBareClassPathResolves`) authors a transient `UWidgetBlueprint` subclassing `UCommonActivatableWidget`, compiles it, then pushes with the bare `<Path>.<Asset>` path against a non-existent host and asserts the error is `HOST_NOT_FOUND` (NOT `CLASS_NOT_FOUND`) — i.e. the bare path passed class resolution + the guard. Reverting to raw `LoadClass` makes it `CLASS_NOT_FOUND` and the test fails. The pre-existing `FUiActivatablePushBadClassTest` (truly bogus path → `CLASS_NOT_FOUND`) is retained.
