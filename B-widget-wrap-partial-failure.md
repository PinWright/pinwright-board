---
id: B-widget-wrap-partial-failure
title: "`widget.wrap` leaves a constructed wrapper or a changed widget hierarchy behind when a later property or attachment step fails"
status: OPEN
severity: High
category: bug
tags: [widget, wrap, transactions, rollback, partial-mutation]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# `widget.wrap` mutates the widget tree before every failure has been ruled out

## What happens

`widget.wrap` opens a transaction and constructs the wrapper before applying
`wrapperProperties` (`WidgetWrapHandler.cpp:153-174`). A bad property returns an
error without discarding that newly constructed widget. The handler then replaces
the target in its old parent and adds it below the wrapper (`:182-195`) before it
applies `wrapperSlot`; a bad slot property returns at `:204-210` with the completed
hierarchy change still in memory. The two attachment failures have the same shape.

`FScopedTransaction` records an undo step but does not reverse changes when the
scope exits. A failed call can therefore leave an orphan source widget or a
partially wrapped tree, and a later save can make that failed request durable.

## Why it matters

The error receipt is not atomic. A retry can hit `DUPLICATE_NAME`, and later widget
work proceeds against a hierarchy the caller believes was rejected. Severity is
High under the board's partial-mutation rule; no durable corruption was reproduced
in this source-only scan.

## What should happen

Preflight both property objects before constructing or reparenting. Also capture a
transactional snapshot and use `ApplyAndCancelTransaction` on every late failure,
plus `DiscardConstructedWidgetForAuthoring` for a wrapper created by this request.
Verify that the original parent, index, slot and target hierarchy are restored.

## Workaround

Call `widget.wrap` without `wrapperProperties` or `wrapperSlot`, then apply those in
separate calls. After any wrap error, undo or reload the asset before retrying.

## Related

- `B-widget-set-partial-application` — established the complete rollback pattern.
- `B-xml-partial-creation` — established explicit cleanup for constructed widgets.
- `F-widget-wrap` — feature ticket; it does not track failed-call atomicity.

## History
- `#1-source-scan-partial-wrap` `OPEN` reporter — Source-only pattern scan confirmed construction at `WidgetWrapHandler.cpp:155`, hierarchy writes at `:182-189`, and error exits at `:172-173`, `:184-186`, `:192-194`, and `:207-210` with no cancellation, snapshot restore, or constructed-widget cleanup. No build, test, editor, MCP call, or plugin edit was performed.
