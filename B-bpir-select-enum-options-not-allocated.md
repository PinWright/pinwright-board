---
id: B-bpir-select-enum-options-not-allocated
title: "BPIR select() allocates only 2 option pins; enum-driven 3+ option selects fail to compile"
status: DONE
severity: Medium
category: bug
tags: [bpir, compiler, select, enum, round-trip, k2node-select]
---

# `select(...)` doesn't allocate option pins for enum index

BPIR `select(...)` instruction always emits a `UK2Node_Select` with the
default 2 option pins (`Option 0`, `Option 1`, plus `Index`). When the
caller wires an enum-typed value into `Index:` and supplies 3+ option
arguments matching the enum entries, the compiler errors:

```
COMPILE_FAILED: Could not find target pin 'Option 2' on node 'Select'.
Available pins: Option 0, Option 1, Index
```

UE's own `UK2Node_Select::AllocateDefaultPins`
(`Engine/Source/Editor/BlueprintGraph/Private/K2Node_Select.cpp:206-262`)
does the right thing when its `Enum` UPROPERTY is set: it allocates
one option pin per `EnumEntry`. BPIR's `select` opcode never sets
`Enum`, so the node falls back to the 2-pin numeric/bool default.

The decompile side already emits the 3+ option form. Sample from
`W_MissionSelectStabilisation::FlyButton.OnButtonBaseClicked`:

```
%n5 = select(Index: %n1.AsBDroneGameInstance.ForcedFlightMode,
             true: %n2,
             false: %n3,
             `Option 2`: %n4)
```

`ForcedFlightMode` is an `EFlightMode` enum with 3 entries (`Acro`,
`AltHold`, `Navigation`). Recompiling this decompile fails because
only Option 0/1 exist on the freshly-emitted node.

Distinct from `B-bpir-select-true-false-swapped` — that ticket was
about the 2-option boolean mapping. This is about >2 option pins not
existing in the first place.

## Repro

1. Decompile any graph that uses `select` indexed by a 3+ entry enum
   (e.g. `W_MissionSelectStabilisation::FlyButton.OnButtonBaseClicked`
   selecting on `EFlightMode`).
2. Feed the BPIR back into `blueprint.compile_bpir`.
3. `COMPILE_FAILED: Could not find target pin 'Option 2'`.

## Impact

- Breaks decompile → edit → compile for any BP using enum-driven
  select with 3+ entries — common pattern for picking strings/textures
  by mode (flight mode, drone class, camera mode, ...).
- Forces a fallback to the generic `call K2Node_Select(...)` syntax
  with `node_props { Enum: /Script/Mod.EEnumName }`, which works but
  is verbose and not how the canonical BPIR `select` opcode should
  be authored.

## Workaround

Replace `select(...)` with the generic K2Node syntax:

```
%n5 = call K2Node_Select(Index: %enumVal, Acro: %a, AltHold: %b, Navigation: %c)
       node_props { Enum: /Script/PDSGame.EFlightMode }
```

The `Enum` property is shape-determining so it's applied before
`AllocateDefaultPins`, which then creates one pin per enum entry with
the enum value names. Compile then succeeds.

## Fix

`BpirCompiler` Select emission should mirror UE's own enum-mode
allocation through public API:

1. Resolve the `Index:` / `Condition:` / `cond:` source at emit time.
2. If the source is `PC_Byte` or `PC_Enum` with a `UEnum*` subcategory,
   pass the enum to `FCodeNodeEmitter::CreateSelectNode`.
3. `CreateSelectNode` calls public `UK2Node_Select::SetEnum(ResolvedEnum, true)`
   before `AllocateDefaultPins()`, so UE creates one option pin per enum entry.
4. Bind enum-backed option args by exact pin name, stripped enum scope, and
   normalized name. Keep `true:` / `false:` binding only for non-enum boolean
   Select nodes.

Decompile side should emit enum-backed Select option pins with their enum
entry names (`Acro: ..., AltHold: ..., Navigation: ...`) instead of the
mixed `true: / false: / Option 2:` form. Boolean `Option 0` / `Option 1`
still decompile as `false:` / `true:` for non-enum Select nodes.

Add a round-trip regression test: BPIR with an `ESlateVisibility`-indexed
select, distinct values for `Visible`, `Collapsed`, and `Hidden`, decompile
→ recompile → assert the enum option pins exist and values land in the
right pins.

## History
- `#1-initial-repro` `OPEN` reporter — Session compile of `W_MissionSelectStabilisation::BP_OnActivated` containing `select(Index: %n1.AsBDroneGameInstance.ForcedFlightMode, \`Option 0\`: %nAcro, \`Option 1\`: %nAlt, \`Option 2\`: %nNav)` failed with `Could not find target pin 'Option 2' on node 'Select'. Available pins: Option 0, Option 1, Index`. Same shape with enum-entry-name pins (`Acro: ..., AltHold: ..., Navigation: ...`) failed identically — the node has only 2 option pins regardless of the args supplied. Worked around by switching to `call K2Node_Select(...) node_props { Enum: /Script/PDSGame.EFlightMode }`, which forces the `Enum` UPROPERTY to be set before `AllocateDefaultPins` and gets the right pin count.
- `#2-select-enum-allocation` `IN-REVIEW` developer — Select compile now resolves enum index sources, calls public `UK2Node_Select::SetEnum(ResolvedEnum, true)` before pin allocation, binds enum option labels by exact/scoped/normalized names, and decompiles enum-backed Select nodes with enum option names; regression test `FSelectEnumAllocatesAllOptionsRoundTripTest` covers `ESlateVisibility` `Visible`/`Collapsed`/`Hidden` option allocation, defaults, decompile labels, and recompile.
- `#3-verify-fix` `DONE` tester — Verified: created temp `W_McpVerifyTemp_select_enum` UserWidget, compiled BPIR `select(Index: $Vis, Visible: "vis", Collapsed: "col", HitTestInvisible: "hti", SelfHitTestInvisible: "shti", Hidden: "hid")` indexed by `ESlateVisibility` (5 entries) — `compiled: true, errors: []`. Decompile round-tripped all 5 enum option pins with their entry names; temp BP deleted.
