---
id: B-bpir-member-call-through-chained-property
title: "BPIR cannot call a member function through a chained property ref (`Target: %m.PauseRef`) — 'Unresolved function', and a same-named local function silently binds Target to self"
status: OPEN
severity: Medium
category: bug
tags: [bpir, compile_bpir, member-call, chained-property, target-pin, function-resolution]
encounters: 1
costly: 1
lastSeen: 2026-09-03T00:50:00Z
---

# A chained property is not usable as a member-call `Target`

## Repro A — `Unresolved function` through a chained ref

```
entry function DebugMenuNav() {
    %c = call GetController(Target: self)
    %m = cast<BP_HUDManager_C>(%c)
    %okp = call IsValid(Object: %m.PauseRef)          <- this resolves fine
    %b2 = branch(%okp) [true -> @nav, false -> @endNav]
@nav:
    call DebugFocusNext(Target: %m.PauseRef)          <- this does not
@endNav:
    return
}
-> [COMPILE_FAILED] Unresolved function: 'DebugFocusNext'. Searched:
     - Blueprint class: BP_HUDTestPawn_C
     - UKismetMathLibrary ... and 335 more libraries
```

`%m.PauseRef` is a `WBP_PauseMenu_C` reference and `DebugFocusNext` is public on that class. The
same chain works as a **value** (`IsValid(Object: %m.PauseRef)` compiles), so the chain resolves;
what fails is using it to pick the class for function lookup. Binding the same value to a typed
member variable first and calling `Target: $MenuRef` works.

## Repro B — same-named local function captures the Target pin

Before hitting A, the pawn's own function was also called `DebugFocusNext`:

```
call DebugFocusNext(Target: %m.PauseRef)
-> [COMPILE_FAILED] TryCreateConnection failed wiring data 'PauseRef' -> 'self'
```

The resolver matched the **pawn's own** `DebugFocusNext` (a self-call whose only object pin is
`self`) and then tried to wire the widget reference into that `self` pin. Renaming the caller to
`DebugMenuNav` changed the error to Repro A, which is how the two were separated.

## What should happen

- A chained property ref should supply its class for member-function resolution wherever it is
  accepted as a value, so `Target: %ref.Prop` works like `Target: $TypedVar`.
- When a name matches both a local function and a member function of the supplied `Target`'s class,
  the `Target`'s class should win — or the error should say the name was ambiguous, rather than
  reporting a pin-wiring failure that points at the argument.

**Workaround:** assign the chained value to a typed member variable and call through that, or move
the call into the class that already owns a typed reference (what I did: the helper now lives on
`BP_HUDManager`, which has `PauseRef` as a typed variable).

severity rationale: impact=soft blocker with a workaround, but both diagnostics misdirect — one
lists 335 libraries without mentioning the chain, the other blames the argument x reach=calling into
a widget or subobject held by another class is routine -> Medium.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) building a pawn-side test hook that drives the pause menu (`BP_HUDTestPawn` -> `BP_HUDManager.PauseRef` -> `WBP_PauseMenu.DebugFocusNext`), needed because `B-simulate-input-key-events-never-reach-pie-pawn` blocks the critic from opening the menu. Repro B came first (identical function names), Repro A after renaming. `IsValid(Object: %m.PauseRef)` on the line above compiles in the same body, which is what shows the chain itself is fine and only the function-lookup path rejects it. Resolved by relocating the helper onto the class holding the typed reference.
