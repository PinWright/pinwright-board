---
id: B-bpir-create-event-self-pin-unresolved
title: "K2Node_CreateDelegate decompile emits 'Create_Event(self: ?)' — self pin unresolved when target is implicit context"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, create-delegate, self-pin]
---

# `Create_Event(self: ?)` — implicit-self resolution missing

The decompiler's typed handler for `K2Node_CreateDelegate` (which
emits `Create_Event(self: <target>, ...)` to wire delegate bindings)
fails to resolve the `self` pin when the binding target is the
**containing object's implicit context** (the most common case in
Widget Blueprints, where a delegate binding to a member function
is implicitly self-bound).

The result is `call Create_Event(self: ?)` — the `?` token at the
named-arg site. The compiler then has nothing to bind to, breaking
round-trip.

**Counts in fresh 2026-05-04 dump:** ~198 occurrences across 74–85
`bpir.txt` files. Almost all in PDS App/UI widgets where buttons,
tabs, and animations bind to member events on the widget itself.

**Examples:**
- `App\App\LevelBlueprints\B_DroneGameMode\bpir.txt`
- `App\App\UI\LobbyAndMenu\Elements\Buttons\W_ImageButtonWithBlurBG\bpir.txt`
- `App\App\UI\LobbyAndMenu\Elements\SpectatorCameraControl\W_StationaryCameraControl\bpir.txt`

7 files DO emit `Create_Event(self: self)` or `Create_Event(self: $VarName)`
correctly, confirming the typed emitter has partial support — only
the implicit-context case fails.

**Distinct from optional-pin omission:** the `self` pin here is
*wired* in the BP graph (to the implicit "self" of the containing
object), not optional-and-unwired. `IsPinOmittableAtCallSite` does
not apply. The fix belongs in the `K2Node_CreateDelegate` typed
emitter's pin resolver: detect the case where the `self` pin's
linked-to source is the containing-object self-context, and emit
`self` literal.

**Fix (proposed):** In the `EmitCreateDelegate` (or equivalent)
emitter:
1. If `SelfPin->LinkedTo[0]->GetOwningNode()` is a `K2Node_Self` →
   emit `self`.
2. If `SelfPin->LinkedTo` is empty AND the `K2Node_CreateDelegate`
   lives inside a member function/event of the same class as the
   delegate signature owner → emit `self` (the engine treats this
   as implicit self-binding).
3. Otherwise resolve via the existing `ResolveInputValue` path
   (which already returns `$VarName` correctly when the binding is
   to a variable).

## Acceptance
- Decompile the cache examples; `: ?` count on `Create_Event` lines
  drops to zero.
- Round-trip: recompiled BP produces an equivalent
  `K2Node_CreateDelegate` with the same self binding.

## History
- `#1-initial-repro` `OPEN` reporter — Fresh 2026-05-04 audit + parity sweep both flagged this as the highest-volume `?` token in BPIR (198/221 surviving `?` occurrences). Distinct from `B-bpir-optional-pin-no-default-question-mark` because the pin is wired to implicit-self, not unwired. Fix is in the typed `K2Node_CreateDelegate` emitter, not the optional-pin predicate.
- `#2-typed-create-delegate-emit` `IN-REVIEW` developer — Added typed `K2Node_CreateDelegate` branch in `BpirTextEmitter.cpp::EmitPureNode` before the generic-pure fallback. Implicit-self (unwired Self pin) now emits literal `self` instead of falling through to `ResolveInputValue` which yields `?`. Wired cases still go through `ResolvePin`. Function name comes from `GetFunctionName()`. Test `FBpirCreateDelegateImplicitSelfEmitsSelfToken` covers the unwired-self case.
- `#3-verify-create-event-self` `DONE` tester — Verified: `blueprint.decompile` on `/App/App/LevelBlueprints/B_DroneGameMode.B_DroneGameMode`, `/App/App/UI/LobbyAndMenu/Elements/Buttons/W_ImageButtonWithBlurBG.W_ImageButtonWithBlurBG`, and `/App/App/UI/LobbyAndMenu/Elements/SpectatorCameraControl/W_StationaryCameraControl.W_StationaryCameraControl` returned `Create_Event(self: self, ...)` and no `Create_Event(self: ?)` occurrences.
