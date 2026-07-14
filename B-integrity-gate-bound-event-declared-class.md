---
id: B-integrity-gate-bound-event-declared-class
title: "Save integrity gate false-positive: ComponentBoundEvent delegate resolved on the DECLARED property class, not the node's DelegateOwnerClass — blocks saving BPs whose C++ base widens a bind type"
status: IN-REVIEW
severity: High
category: bug
tags: [integrity-gate, asset-save, component-bound-event, delegate, bind-widget, false-positive]
encounters: 1
lastSeen: 2026-07-14T18:30:00Z
---

# Integrity gate resolves bound-event delegates on the declared property class → false-positive save block

## Symptom (live incident)

`asset.save` on `W_HUD_Common` (freshly reparented to a C++ base) returned
`integrityGate:"blocked"` with 4 failures:
`K2Node_ComponentBoundEvent ... does not resolve to an FMulticastDelegateProperty on component
class 'UserWidget'` — one per bound click event on the embedded `W_CommonTrackControls` widget.
The blueprint compiled `errors: [] warnings: []` and the engine resolves those events fine at
runtime. No bypass exists (the gate fires in `asset.save`, `editor.save_all`, and
`SaveLoadedAssetThrottled`), so a perfectly valid asset became unsaveable and the session's
in-memory work was stranded.

## Root cause

`BlueprintHandlerUtils.cpp` (~:2126-2145): the checker resolves the delegate by looking up
`DelegatePropertyName` on `CompProp->PropertyClass` — the **declared** type of the component
property on the generated class. After a C++ base takes over a widget bind as a widened type
(`UPROPERTY(meta=(BindWidgetOptional)) TObjectPtr<UUserWidget> W_CommonTrackControls;` — the
embedded widget is BP-only, so no tighter C++ type exists), the declared class is `UUserWidget`,
which has no such dispatcher. The **engine** resolves via the node's serialized
`DelegateOwnerClass` (`K2Node_ComponentBoundEvent::GetTargetDelegateProperty`), which still points
at `W_CommonTrackControls_C` — where the dispatcher lives.

The pattern (C++ base declares a widened bind; BP instance carries the dispatchers) is standard
and increasingly common in the host project's HUD refactor; every such widget would hit this gate.

## Fix applied (this commit)

In the ComponentBoundEvent branch: when the delegate is not found on the declared property class,
fall back to `BoundEvent->DelegateOwnerClass` before recording a failure — mirroring the engine's
own resolution order. Failure message updated to say both classes were checked.

## Verification for the tester

1. Reparent any widget BP with component-bound events on an embedded BP widget onto a C++ base that
   declares the embedded widget as `BindWidgetOptional TObjectPtr<UUserWidget>`.
2. Compile (clean) → `asset.save`: previously `integrityGate:"blocked"` with the
   FMulticastDelegateProperty failure; with the fix, saves cleanly.
3. Negative check intact: a bound event whose delegate exists on NEITHER class (genuinely broken
   asset) must still block.

## History

- `#1-filed-and-fixed` **IN-REVIEW** (Developer) — Filed from the W_HUD_Common save block during
  the HUD refactor epilogue; one-branch fix (DelegateOwnerClass fallback) applied in
  `BlueprintHandlerUtils.cpp` in the same session. Needs tester verification per the steps above.
