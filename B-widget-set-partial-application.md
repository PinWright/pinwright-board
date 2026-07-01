---
id: B-widget-set-partial-application
title: "`widget.set` partially applies properties before returning an error"
status: DONE
severity: High
category: bug
tags: [widget-set, transactions, rollback, validation]
---

# `widget.set` partially applies properties before returning an error

`widget.set` can mutate some requested properties before returning an error on a later property in the same request. This makes failed calls unsafe to retry or reason about: the response says the operation failed, but the asset has already been partially changed in memory.

Observed during montage pause-menu UMG work on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu`. A `widget.set` call attempted to set multiple properties on `W_HotkeyInfo_MontageWheel`, including:

```json
{
  "Description": "Timeline: seek 5 seconds",
  "TextMid": "[Wheel]",
  "TextLeft": "",
  "TextMidLS": "",
  "TextRight": "",
  "KeyImage": "None"
}
```

The call returned:

```text
INVALID_PROPERTY: Failed to set 'KeyImage': Failed to load object at path: None
```

However, a follow-up `widget.export_xml` showed the earlier properties from the failed request had already been applied:

```xml
<W_HotkeyInfo-Image_C name="W_HotkeyInfo_MontageWheel"
  Description="Timeline: seek 5 seconds"
  KeyImage="/App/Textures/UI/HelpScreenIcons/new/RMB.RMB"
  TextMid="[Wheel]" />
```

Expected behavior:

- Either validate every property and slot update before mutating anything, then apply the whole batch only if all updates are valid.
- Or wrap the mutation in a rollback path so any failure restores the widget to its pre-call state before returning an error.
- The response should not report failure while leaving partial authored changes behind unless it explicitly reports `partialApplied: true` and lists exactly which fields changed. The safer default should be atomic all-or-nothing behavior.

Impact: callers cannot safely submit multi-property updates. They must split calls into one property at a time or re-read/export the widget after every failed call to discover what changed.

## History

- `#1-initial-repro` `OPEN` reporter -- During montage pause-menu UMG authoring, `widget.set` returned `INVALID_PROPERTY` for invalid `KeyImage: "None"` but had already applied earlier fields from the same request (`Description`, `TextMid`). Follow-up `widget.export_xml` confirmed the failed call left partial changes in the widget tree.
- `#2-rollback-on-error` `IN-REVIEW` developer -- Changed WidgetSetHandler.cpp to snapshot widget/slot state before applying a widget.set batch and roll back the transaction before returning any mid-batch error; added FWidgetSetRollsBackOnSlotErrorTest to prove earlier property writes do not persist when a later slot property is invalid.
- `#3-verified-rollback-live` `DONE` tester — Verified live on duplicated temp widget `/Game/McpVerify/W_McpVerifyTemp.W_McpVerifyTemp`: `widget.set` on `Time` with `properties.RenderOpacity:0.25` plus invalid `slot.NotARealSlotProperty` returned `INVALID_PROPERTY`, and follow-up `widget.describe include_defaults:true include_slot:true` still showed `RenderOpacity: 1`, so the earlier property write was rolled back.
- `#4-regression-prior-add-undone` `OPEN` reporter — Regression of the `#2` rollback fix: `widget.set`'s `ApplyAndCancelTransaction` (TransactionUtils.cpp `GEditor->Trans->Undo(false)`) over-reaches across RPC calls and undoes a prior unsaved `widget.add` from a separate call. Repro on `/App/App/UI/W_PhotoPopup` (no `editor.save_all` between calls): (1) `widget.add` `type:TextBlock name:ReasonText parentName:Card` → success (widget count 12); (2) `widget.set` widgetName `ReasonText` with `properties.Text: "INVTEXT(\"\")"` → `INVALID_PROPERTY: Failed to set 'Text': Persisted FText values require a non-empty namespace and key`; (3) follow-up `widget.set` widgetName `ReasonText` with `properties.Text: NSLOCTEXT(...)` → `NOT_FOUND: Widget not found: ReasonText`; (4) `widget.describe` reports 11 widgets, ReasonText gone; (5) re-running the same `widget.add` succeeds and persists. Hypothesis: the FText validation in `ApplyJsonValueToProperty` rejects the value before mutating the widget, so the `widget.set` transaction (opened with `PrepareTransactionalSnapshot(Widget)`) ends up empty/diff-less; `UTransBuffer::Undo(false)` then skips the empty record and unwinds the previous transaction on the buffer — the `widget.add` `FScopedTransaction` from the prior RPC call — reverting the widget construction. The `FWidgetSetRollsBackOnSlotErrorTest` test only exercises within-call rollback and does not cover this cross-call case. **Workaround:** call `editor.save_all` after every `widget.add` before doing any `widget.set` that might fail, or recreate the widget after any `widget.set` error. **Fix options:** (a) before `ApplyAndCancelTransaction`, snapshot the active transaction's record count and refuse to undo if the just-ended transaction recorded zero diffs (treat as no-op cancel instead of `Undo`); (b) replace `GEditor->Trans->Undo(false)` with a targeted `FTransaction::Apply` against the specific transaction object we just ended, instead of trusting the buffer top; (c) keep widget.set's transaction non-empty by always calling `Modify()` on the WidgetTree (so empty-diff fallthrough cannot happen) — but (a) is the least invasive.
- `#5-guard-empty-rollback` `IN-REVIEW` developer -- Guarded `ApplyAndCancelTransaction` so it only calls `Undo(false)` when the just-ended transaction remains at the undo-buffer top; added `FWidgetSetFailurePreservesPriorAddTest` to prove failed empty `widget.set` preserves a prior `widget.add`.
- `#6-review-scope-widget-fixtures` `IN-REVIEW` developer -- Documented that `FWidgetSetFailurePreservesPriorAddTest` depends on the shared `Tests/Widget/WidgetTestFixtures.h` transient Widget Blueprint helpers, so the helper header is part of the reviewed regression-test scope.
- `#7-verified-cross-call-add-preserved` `DONE` tester — Verified on temp `/Game/App/UI/Test/W_McpVerifyTemp_widget_set_partial`: created widget BP, `widget.add` TextBlock `ReasonText` to `RootCanvas` (no save_all), then `widget.set` `Text:"INVTEXT(\"\")"` returned `INVALID_PROPERTY` ("Persisted FText values require a non-empty namespace and key"); follow-up `widget.describe` still showed `widget_count: 2` with `ReasonText` present under `RootCanvas`, confirming the empty-diff guard no longer undoes the prior `widget.add`. Temp asset deleted.
