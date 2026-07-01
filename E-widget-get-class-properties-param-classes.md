---
id: E-widget-get-class-properties-param-classes
title: "widget.get_class_properties requires `classes`, but the sibling class-name-array slot asset.search_assets uses `classNames[]` — the natural guess hard-errors MISSING_REQUIRED_PARAM with no alias"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, widget, asset, param-alias, classes, classNames, get_class_properties, search_assets, misuse-then-correct, drift]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# `widget.get_class_properties` takes `classes`; the class-name-array slot drifts from `asset.search_assets`'s `classNames[]`

New instance of the param-name-guessability family
(`E-class-name-format-inconsistency` DONE — class-name *format* drift;
`E-widget-remove-widget-param-name` DONE — widget-instance selector;
`E-widget-asset-path-alias-drift` DONE — asset-path slot;
`E-effect-actor-name-slot-vs-actorname` OPEN — actor-name slot). None of those
covers the **array-of-class-names selector**, which is the slot that drifts here.

Two MCP verbs take "a list of UMG/engine class names" as an array, with two
different param keys and no alias bridging them:

- **`asset.search_assets`** — `classNames[]` (`AssetQueryHandler.cpp`, confirmed
  in `E-class-name-format-inconsistency` table and `E-asset-search-vs-search-assets-overlap`).
- **`widget.get_class_properties`** — `classes` (array), **required**, no
  `classNames` alias. Source:
  `Source/PinWright/Private/Handlers/UI/WidgetClassInspectHandler.cpp:49-59`:
  `RPC_PARAM_REQ("classes", "array", "UMG class names, e.g. [\"Button\", \"TextBlock\"]")`,
  read via `Ctx.GetArray(TEXT("classes"))`, with a manual
  `MISSING_PARAMETER` ("Required parameter 'classes' is missing or empty")
  when absent.

So an agent that has just been (or will soon be) listing assets with
`asset.search_assets { classNames:[...] }` naturally reaches for `classNames`
when it wants the property schema for those same classes via
`widget.get_class_properties`, and eats a hard `MISSING_REQUIRED_PARAM` before
correcting to `classes`. CLAUDE.md's "accept camelCase + snake_case aliases"
rule does not cover this — `classes` and `classNames` are distinct words, not
casing variants — so the wrong guess is a hard error, not a silent accept.

## Repro (verbatim, from the audited WBP_PauseMenu layout task)

The task (namespace `widget`, outcome `clean`) built `/Game/UI/WBP_PauseMenu`
and probed slot-class schemas before authoring the canvas/box layout. Call log,
adjacent calls:

1. `widget.get_class_properties { classNames:[CanvasPanelSlot,VerticalBoxSlot] }`
   → `ok:false`, `error_text:"[MISSING_REQUIRED_PARAM] Missing required
   parameter 'classes' (type: array)"`.
2. retry `widget.get_class_properties { classes:[CanvasPanelSlot,VerticalBoxSlot] }`
   → `ok:true`.

The error is accurate (not misleading) and self-correcting — one wasted
`is_error` call, zero blocked progress — exactly the shape of the precedent
param-drift tickets. Friction note (verbatim): *"widget.get_class_properties
rejected the guessed key 'classNames' and required 'classes' (its wiki page
wasn't read first — the error message corrected it on retry)."*

## What it should do

Reuse the dispatcher `FParamSpec` alias machinery from
`E-blueprint-param-name-path-vs-assetpath #4` / `E-widget-remove-widget-param-name #3`
(the change that lets alias-only required params validate at the wire level).
Standardize the alias *set*, not the canonical name, so existing callers keep
working:

- **`widget.get_class_properties` `classes` slot**
  (`WidgetClassInspectHandler.cpp:52`): add `classNames` (and the snake
  `class_names`) as accepted aliases, reading via a first-of lookup
  (`GetArray` over `{classes, classNames, class_names}`). This makes the two
  class-name-array verbs interchangeable on the array key.

Either canonical is fine; the point is that the same conceptual "list of class
names" should not be `classNames` on one verb and `classes` on the adjacent one
with no bridge.

## Docs angle (`docs/wiki-src/widget.md`)

`widget.get_class_properties` has no method H3 in the widget overlay
(`docs/wiki-src/widget.md` references it only as "much cheaper than iterating
`widget.get_class_properties` per child" inside the `widget.describe` section);
its param key is discoverable only from the live `widget.get_class_properties?`
schema. A short `### widget.get_class_properties` section naming the `classes`
array param (and noting it is the schema-probe verb for slot classes like
`CanvasPanelSlot` / `VerticalBoxSlot`) would close the discovery half until the
alias lands. The wiki edit is a downstream process, not this ticket.

## Not a duplicate of

- `E-class-name-format-inconsistency` (DONE) — that unified class-name *string
  format* (short vs `/Script/` vs BP `_C`) across 5 verbs via `ResolveUClass`;
  it does not touch the *param key* drift between `classes` and `classNames`
  (its own table lists `asset.search_assets`'s `classNames[]` but never
  `widget.get_class_properties`).
- `E-widget-remove-widget-param-name` / `E-widget-asset-path-alias-drift`
  (DONE) — the widget-instance-name and asset-path slots; neither is the
  class-name-array slot.
- `E-asset-search-vs-search-assets-overlap` (IN-REVIEW) — verb-choice overlap
  inside `asset.*`; unrelated to the `classes`/`classNames` array-key drift.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the WBP_PauseMenu UMG
  layout task (namespace `widget`, outcome `clean`, no judge filing). Distinct
  PROCESS angle: one misuse-then-correct round-trip —
  `widget.get_class_properties { classNames:[...] }` →
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'classes' (type: array)`,
  corrected on retry with `{ classes:[...] }`. Root cause is param-key drift on
  the array-of-class-names selector: `widget.get_class_properties` requires
  `classes` (`WidgetClassInspectHandler.cpp:52`, no alias) while the sibling
  class-name-array verb `asset.search_assets` uses `classNames[]`. New slot in
  the param-name-drift family — the existing DONE tickets cover class-name
  format, widget-instance name, and asset-path, but not the class-name *array*
  key. Fix: dispatcher `FParamSpec` alias adding `classNames` (+ `class_names`)
  to `widget.get_class_properties`'s `classes` slot; plus a
  `### widget.get_class_properties` H3 in `docs/wiki-src/widget.md` naming the
  `classes` param.
