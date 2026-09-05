---
id: B-widget-wrap-partial-failure
title: "`widget.wrap` leaves a constructed wrapper or a changed widget hierarchy behind when a later property or attachment step fails"
status: IN-REVIEW
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

## Fix

`widget.wrap` now validates both optional property batches before construction, snapshots the
widget-tree owners and original slot before mutation, verifies the final hierarchy, and rolls back
the transaction while discarding only the newly constructed wrapper on any late failure. The
failed-call contract is atomic. `wrapperProperties.Slots` additionally accepts null entries and
slots whose content belongs to the wrapped target subtree, while refusing foreign widget/slot
references with `INVALID_PROPERTY` before construction; live verification remains outstanding.

## Workaround

Call `widget.wrap` without `wrapperProperties` or `wrapperSlot`, then apply those in
separate calls. After any wrap error, undo or reload the asset before retrying.

## Related

- `B-widget-set-partial-application` — established the complete rollback pattern.
- `B-xml-partial-creation` — established explicit cleanup for constructed widgets.
- `F-widget-wrap` — feature ticket; it does not track failed-call atomicity.

## History
- `#1-source-scan-partial-wrap` `OPEN` reporter — Source-only pattern scan confirmed construction at `WidgetWrapHandler.cpp:155`, hierarchy writes at `:182-189`, and error exits at `:172-173`, `:184-186`, `:192-194`, and `:207-210` with no cancellation, snapshot restore, or constructed-widget cleanup. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-atomic-wrap-rollback` `IN-REVIEW` developer — Changed `Source/PinWright/Private/Handlers/UI/WidgetWrapHandler.cpp` to preflight `wrapperProperties` and `wrapperSlot` on scratch instances, snapshot the widget-tree owners and original slot with `PrepareTransactionalSnapshot`, verify the resulting hierarchy, and apply-and-cancel the transaction plus `DiscardConstructedWidgetForAuthoring` on every late failure. Added handler-level regression `PinWright.widget.wrap.FailurePreservesHierarchy` in `Source/PinWright/Private/Tests/WidgetXml/TestWidgetWrapHandler.cpp`; documented the no-partial-wrap contract in `Docs/wiki-src/widget.md`. Static diff review only; no Unreal/editor/MCP/test run performed.
- `#3-late-add-rollback-ratchet` `IN-REVIEW` developer — Extended `PinWright.widget.wrap.FailurePreservesHierarchy` with a reflected `Slots` seed that makes a `Button` wrapper's `AddChild` fail after `ReplaceChild` has already rehomed the grandparent slot; the case asserts `INTERNAL_ERROR`, no wrapper, and restoration of the original parent, index, slot object, position, and size. Hardened rollback cleanup to clear any remaining wrapper slots before undo so discard cannot recurse into restored original widgets. Scoped diff check only; no Unreal/editor/MCP/test run performed.
- `#4-reject-foreign-slot-references` `IN-REVIEW` developer — Added pre-mutation `wrapperProperties.Slots` validation in `Source/PinWright/Private/Handlers/UI/WidgetWrapHandler.cpp`: null entries and target-subtree content remain supported, while foreign widget/slot content is refused with `INVALID_PROPERTY`. Extended `PinWright.widget.wrap.FailurePreservesHierarchy` to prove an external slot reference leaves both the target and foreign hierarchies unchanged; updated `Docs/wiki-src/widget.md` with the contract. Scoped diff check only; no Unreal/editor/MCP/test run performed.
