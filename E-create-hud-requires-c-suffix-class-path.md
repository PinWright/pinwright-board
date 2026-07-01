---
id: E-create-hud-requires-c-suffix-class-path
title: "`ui.create_hud` rejects the bare Blueprint asset path and silently requires the undocumented `_C` generated-class suffix on widgetPath"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [ui, create_hud, class-resolution, docs]
---

# `ui.create_hud` rejects the bare asset path, requiring an undocumented `_C` suffix

`ui.create_hud {widgetPath:"/Game/Global/Blueprints/WBP_PlayerHUD.WBP_PlayerHUD"}`
— the bare `<Path>.<Asset>` form a caller naturally derives from `widget.describe`
and from every other path-shaped param in the API — fails with:

```
[CLASS_NOT_FOUND] Failed to load widget class: /Game/Global/Blueprints/WBP_PlayerHUD.WBP_PlayerHUD
```

The call only succeeds once the caller appends the generated-class suffix
`_C` (`...WBP_PlayerHUD.WBP_PlayerHUD_C`). Nothing in the `ui.create_hud` wiki
section, the param name, or the error text tells the caller this — the error
just says the class failed to load, which reads like the asset is missing
rather than "you gave me an asset path where I wanted a generated-class path."
The caller is forced into a trial-and-error retry (one `is_error` CLASS_NOT_FOUND,
then the `_C` variant) to discover the required form.

## Why this is awkward (resolver-consistency gap)

`E-class-name-format-inconsistency` (DONE) already built the canonical
`ClassUtils.cpp::ResolveUClass` helper whose stated contract accepts **all** of:
short name, `U`/`A`-prefixed name, full `/Script/<Module>.<Class>` path, and the
`/Game/<Path>/<Blueprint>.<Blueprint>_C` BP generated-class path — and which, per
that ticket's `#2`, "tries direct load, `_C`-appended load, then
`LoadObject<UBlueprint>` → `GeneratedClass`." That helper deliberately appends
`_C` itself so callers don't have to. `ui.create_hud`'s `widgetPath` resolution
was **not** part of that sweep (which touched `inspect_class`,
`widget.create_widget_blueprint`, and `asset.search_assets`), so it still does a
raw `LoadClass`/`LoadObject<UClass>` on the literal string and rejects the bare
asset path. Routing `ui.create_hud` through `ResolveUClass` would make the bare
`.WBP_PlayerHUD` path work and align it with the rest of the API.

## Evidence (this task)

From the audited task's call-log and friction note:

> "ui.create_hud rejected the documented asset path with CLASS_NOT_FOUND and
> silently required the undocumented _C generated-class suffix."

Call sequence:
1. `ui.create_hud {widgetPath: "...WBP_PlayerHUD.WBP_PlayerHUD"}` (bare asset path)
   → `is_error` `[CLASS_NOT_FOUND] Failed to load widget class:
   /Game/Global/Blueprints/WBP_PlayerHUD.WBP_PlayerHUD`
2. `ui.create_hud {widgetPath: "...WBP_PlayerHUD.WBP_PlayerHUD_C"}` (with `_C`)
   → success.

This is a process retry distinct from the wrong-instance write bug filed as
`B-set-widget-text-hits-wrong-instance` — that ticket is about the *write*
landing on the wrong widget; this is about the *load* rejecting the standard
asset path before the caller ever gets a HUD.

## What it should do

- Route `ui.create_hud`'s `widgetPath` through `ResolveUClass` (or equivalent
  `_C`-appending logic) so the bare `/Game/<Path>/<Blueprint>.<Blueprint>` path
  resolves to the generated class, matching `widget.add`'s `type` slot and the
  resolver contract from `E-class-name-format-inconsistency`.
- Until/unless the resolver lands, the `ui.create_hud` wiki overlay
  (`docs/wiki-src/ui.md`, the `### ui.create_hud` section) should explicitly
  state that `widgetPath` is a **generated-class** path requiring the `_C`
  suffix (`/Game/.../WBP_PlayerHUD.WBP_PlayerHUD_C`), and the `CLASS_NOT_FOUND`
  error text should hint "append `_C` for a Blueprint widget" when the bare
  asset path was given.

**Workaround:** Append `_C` to the asset path passed to `ui.create_hud`.

## History
- `#2-route-create-hud-through-resolver` `IN-REVIEW` developer — Root-cause fix: `ui.create_hud` now routes `widgetPath` through the canonical `ResolveUClass` instead of a raw `LoadClass<UUserWidget>(nullptr, *WidgetPath)`, so the bare `/Game/.../WBP.WBP` asset path (and short names) resolve to the generated UClass without the `_C` suffix — closing the resolver-consistency gap left by `E-class-name-format-inconsistency`. Added an `IsChildOf(UUserWidget)` guard so the downstream `CreateWidget<UUserWidget>` cast stays type-safe (ResolveUClass returns any `UClass*`). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/UI/UiHandler.cpp` (handler swap), `Docs/wiki-src/ui.md` (`### ui.create_hud` now documents that `widgetPath` accepts the bare path / `_C` optional). Regression test: `EditorAutomationRpcGateway.ui.create_hud.BarePathResolvesGeneratedClass` in `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestUIHandlers.cpp` — creates a real WidgetBlueprint via the handler then asserts `ResolveUClass` on the bare (`_C`-less) object path resolves to a `UUserWidget` subclass; would fail (null) under the old raw-`LoadClass` path. (`ui.create_hud` needs PIE/`GameViewport` so it can't run end-to-end headlessly; the test exercises the exact resolution step the handler now performs.)
- `#1-initial-audit` `OPEN` reporter — Struggle-auditor finding from a `ui.screenshot` HUD-reference task. `ui.create_hud {widgetPath:"/Game/Global/Blueprints/WBP_PlayerHUD.WBP_PlayerHUD"}` → `[CLASS_NOT_FOUND] Failed to load widget class: ...WBP_PlayerHUD` (1 is_error), then the `...WBP_PlayerHUD_C` variant succeeded — a trial-and-error retry the wiki/param/error never warned about. Resolver-consistency gap: `E-class-name-format-inconsistency` (DONE) built `ClassUtils.cpp::ResolveUClass` to accept the bare `/Game/...Blueprint.Blueprint` BP path (it appends `_C` itself) and routed `inspect_class` / `widget.create_widget_blueprint` / `asset.search_assets` through it, but `ui.create_hud`'s `widgetPath` was not in that sweep. Distinct PROCESS angle from `B-set-widget-text-hits-wrong-instance` (the write-to-wrong-instance bug). Wiki overlay to improve: `docs/wiki-src/ui.md` `### ui.create_hud`.
