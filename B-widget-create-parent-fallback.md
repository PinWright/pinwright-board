---
id: B-widget-create-parent-fallback
title: "`widget.create_widget_blueprint` silently falls back to UUserWidget when an explicit parentClass is missing or incompatible"
status: OPEN
severity: High
category: bug
tags: [widget, create, parent-class, parameter-validation, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# An invalid requested widget parent produces a different asset successfully

## What happens

`widget.create_widget_blueprint` defaults `ParentUClass` to `UUserWidget`. For an
explicit non-default `parentClass`, it replaces that default only when
`ResolveUClass` succeeds and the class derives from `UUserWidget`; otherwise it
silently keeps the default (`WidgetCreateHandler.cpp:84`, `:122-131`). It then
creates and saves the blueprint and returns success (`:133-189`).

The class check also occurs after `CreatePackage` at `:113-120`, so there is no
preflight boundary at which a bad explicit parent can be refused cleanly. A typo or
an incompatible class such as `Actor` creates a valid but semantically different
UserWidget asset at the requested path.

## Why it matters

The asset exists and the response is green, so automation can author a complete UI
against the wrong base class before the missing behavior is discovered. Severity is
High for a silently dropped authoring parameter.

## What should happen

Resolve and type-check every explicit `parentClass` before creating the package.
Return `INVALID_PARENT_CLASS` with resolution/type details on failure. On success,
read back and return the actual parent class path from the created blueprint.

## Workaround

Resolve the class separately before creation and inspect the created blueprint's
parent class before adding widgets or graph logic.

## History
- `#1-source-scan-parent-fallback` `OPEN` reporter — Source-only scan confirmed invalid and incompatible explicit parent values leave `ParentUClass` at `UUserWidget` and still flow through asset creation and success. No build, test, editor, MCP call, or plugin edit was performed.
