---
id: B-widget-create-parent-fallback
title: "`widget.create_widget_blueprint` silently falls back to UUserWidget when an explicit parentClass is missing or incompatible"
status: IN-REVIEW
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
Return `CLASS_NOT_FOUND` for an unresolved value or `CLASS_NOT_INSTANTIABLE` for
a resolved but incompatible/non-instantiable value, with resolution/type details
on failure. On success, read back and return the actual parent class path from the
created blueprint.

## Workaround

Resolve the class separately before creation and inspect the created blueprint's
parent class before adding widgets or graph logic.

## Fix

Root cause: parent resolution ran after `CreatePackage`, and failed or
incompatible explicit values left the default `UUserWidget` selected. The handler
now resolves and validates every explicit parent before package creation, returns
typed `CLASS_NOT_FOUND` or `CLASS_NOT_INSTANTIABLE` errors, and reports the
created Blueprint's actual `parentClass` path. No invalid explicit parent falls
back to `UUserWidget`; the documented canonical root remains allowed even though
the engine marks that root abstract.

Per the user brief, this supersedes the old `INVALID_PARENT_CLASS` contract:
unresolved values use `CLASS_NOT_FOUND`, while resolved incompatible or
non-instantiable values use `CLASS_NOT_INSTANTIABLE`.

Files:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/UI/WidgetCreateHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Media/TestUIHandlers.cpp`
- `Plugins/PinWright/Docs/wiki-src/widget.md`

Test IDs:

- `PinWright.widget.create_widget_blueprint.InvalidParentRefused` — checks the
  typed unresolved-parent error and verifies no package, registry asset, or file.
- `PinWright.widget.create_widget_blueprint.IncompatibleParentRefused` — checks
  the typed resolved-but-incompatible error and verifies no package, registry
  asset, or file.
- `PinWright.widget.create_widget_blueprint.SaveWritesToDisk` — checks the
  successful response includes the actual parent path while proving disk and
  registry visibility.

Deliberately unchanged: the omitted-parent default remains `UUserWidget`, the
existing name/folder path guards remain in place, and no already-created assets
are migrated or deleted. No build, test, editor, or MCP execution was performed;
this ticket is source-reviewed.

## History
- `#1-source-scan-parent-fallback` `OPEN` reporter — Source-only scan confirmed invalid and incompatible explicit parent values leave `ParentUClass` at `UUserWidget` and still flow through asset creation and success. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-typed-widget-parent` `IN-REVIEW` developer — Moved parent preflight ahead of package creation, added typed errors and actual-parent reporting, and added no-asset regression coverage. No build, test, editor, or MCP execution was performed.
