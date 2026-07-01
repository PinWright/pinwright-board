---
id: B-bpir-field-notification-delegate-pin-wire-fails
title: "BPIR cannot emit FieldNotification subscription — `Delegate` pin wire fails"
status: WONTFIX
severity: Medium
category: bug
tags: [bpir, field-notification, delegate-pin, k2-add-field-value-changed-delegate, create-delegate, make-delegate]
---

# BPIR cannot emit FieldNotification subscription

Emitting the standard FieldNotification subscription pattern via BPIR fails at the delegate-pin wiring step. The decompiler can read this pattern out of an existing BP (it round-trips as `call K2_AddFieldValueChangedDelegate(FieldId: (FieldName="X"), Delegate: $OutputDelegate)`), but the compiler can't re-emit it from that exact form.

## Repro (observed this session, `W_RenameReplay` Construct override)

Trying to retarget an inherited field-notification binding from the deleted `CustomTrackInfo` variable to the new `Replay` variable:

```
entry override Construct() {
    call K2_AddFieldValueChangedDelegate(FieldId: (FieldName="Replay"), Delegate: $OutputDelegate)
}

entry override Destruct() {
    call K2_RemoveFieldValueChangedDelegate(FieldId: (FieldName="Replay"), Delegate: $OutputDelegate)
}
```

Result: `COMPILE_FAILED: Line 18: TryCreateConnection failed wiring data 'OutputDelegate' -> 'Delegate'; Line 22: TryCreateConnection failed wiring data 'OutputDelegate' -> 'Delegate'`.

`$OutputDelegate` is a synthetic local-name that the decompiler emits to represent the output of a `MakeFieldValueChangedDelegate` node connected to a `K2Node_CreateDelegate` bound to the local `SetValues` custom event. The compiler has no recipe to *create* that pair of nodes from the textual `$OutputDelegate` reference, so the `K2_AddFieldValueChangedDelegate` call's `Delegate` input pin gets nothing wired to it.

The original `W_RenameTrack` clone had the binding in place wired correctly (CreateDelegate → SetValues + MakeFieldValueChangedDelegate already authored in the editor). Removing those overrides at session-end was acceptable because the popup is one-shot and the caller pre-populates `Replay` + calls `ApplyReplay` directly from the spawn site. So in practice the workaround was "skip field-notification entirely and trigger the update path inline."

## Impact

Any BPIR-driven port of a BP that subscribes to field-notify (the canonical UE 5.4+ pattern for reactive UMG) loses the reactive subscription. The author has two choices:
1. Skip field-notify, push state explicitly from the spawn site (the workaround used this session).
2. Keep the inherited binding pointing at a stale variable name (FieldName is just a string at compile time — won't fail compile, won't fire at runtime).

Neither is suitable for a BP that reactively reflects late-arriving field changes (the whole point of FieldNotification).

**Workaround (this session):** have the spawn site explicitly call the popup's `ApplyXxx(Target: %popup, ...)` immediately after setting the data var. Bypasses the subscription path entirely.

**Proposal:** add a BPIR primitive for FieldNotification subscription, e.g.:
```
field_notify_subscribe(FieldName: "Replay", Handler: @SetValues)
field_notify_unsubscribe(FieldName: "Replay", Handler: @SetValues)
```
The compiler synthesizes the `MakeFieldValueChangedDelegate` + `CreateDelegate` + `K2_AddFieldValueChangedDelegate` triple. Or, less invasively, recognize the decompiler's own `$OutputDelegate` round-trip form and emit the supporting nodes when seen as the `Delegate:` argument to `K2_(Add|Remove)FieldValueChangedDelegate`.

Related but distinct from `B-bpir-bind-dispatcher-external-target-local-event` — that ticket is about cross-object dispatcher binding via `bind_dispatcher`; this is the FieldNotification flavor which uses a different node trio.

## History
- `#1-initial-repro` `OPEN` reporter — Hit on `W_RenameReplay` clone of `W_RenameTrack`. Both `K2_AddFieldValueChangedDelegate` and `K2_RemoveFieldValueChangedDelegate` calls fail identically with `TryCreateConnection failed wiring data 'OutputDelegate' -> 'Delegate'`. Worked around by removing the Construct/Destruct overrides entirely and pushing state from the row-spawn site.
- `#2-reclassified-as-feature` `WONTFIX` developer — Reclassified to `F-bpir-field-notify-primitive`. Sprint investigation confirmed no pre-existing decompiler round-trip contract for `$OutputDelegate`: it is an accidental rendering from the generic pure-node fallback in `BpirDecompiler.cpp:1334-1355`, not an intentional emission shape. The compiler's `PreEmitDollarVar` (`BpirValueResolver.cpp:285-331`) also has no special-case synthesis for this marker — it just fails a variable lookup, yielding the observed `TryCreateConnection failed` error. Fix path is a new first-class primitive analogous to `bind_dispatcher`, tracked in `F-bpir-field-notify-primitive`. Original repro carries over to the new ticket.
