---
id: B-bpir-pc-fieldpath-absent
title: "BPIR type system has no entry for PC_FieldPath / TFieldPath<FProperty> — UE 5.3+ MVVM and field-bind nodes unrepresentable"
status: DONE
severity: Medium
category: bug
tags: [bpir, type-system, pc-fieldpath, mvvm]
---

# `PC_FieldPath` is not in the BPIR type system

UE pin category `PC_FieldPath` (used for `TFieldPath<FProperty>` —
a serializable reference to a UPROPERTY by path) has:
- No `EBpirTypeKind` variant.
- No entry in `BpirTypeGrammar.cpp` (`GetTable()`).
- No branch in `FindByPinCategory` — the switch falls off after
  `PC_MCDelegate`, returning null for any FieldPath pin.

Any Blueprint with a `TFieldPath<FProperty>` pin (function param,
variable, or node arg) hits this hole. The decompiler reaches the
terminal `return DefaultValue` branch in `ResolveInputValue`,
emitting the raw UE format string (or `?` if unset). The compiler's
`BpirTypeSpecParser::ParseTypeSpec` cannot recognize the type
back, so round-trip fails.

## When does it show up

UE 5.3 introduced MVVM (Model-View-ViewModel) and a set of
ViewModel/View binding nodes that route through `TFieldPath` pins
to identify which UPROPERTY to bind. The PDS project may not use
MVVM directly today, but:
- Engine `K2Node_PromotableOperator` and certain reflection nodes
  use FieldPath internally.
- Plugins that integrate with MVVM (CommonUI extensions, viewmodel
  plugins) use FieldPath pins on their delegate signatures.
- The `K2Node_FieldNotificationGetField` and related field-notify
  helpers expose FieldPath args.

Empty PDS-side hits today does not mean unaffected — the moment a
BP uses one of these node types, BPIR breaks silently.

The existing `field_notify_subscribe` / `field_notify_unsubscribe`
sugar uses `struct<FieldNotificationId>` (PC_Struct), not
PC_FieldPath, so field-notify itself is unaffected. This is about
generic FieldPath pins.

## Fix

- Add a bare BPIR type `field_path` for `UEdGraphSchema_K2::PC_FieldPath`.
- Accept `fieldpath` as a legacy alias for the raw UE category currently emitted by fallback paths.
- Add `FindByPinCategory` routing for `PC_FieldPath`.
- Keep container support as `array<field_path>`, not `array<field_path<C.P>>`.
- Treat FieldPath defaults as string values during decompile/compile.

## Repro

Find a UE 5.3+ MVVM widget BP and decompile. Or, for a synthetic
repro, add a `TFieldPath<FProperty>` UPROPERTY to a custom BP
function library and use it via a Blueprint.

## History
- `#1-initial-spec` `OPEN` reporter — Type-system parity audit found `PC_FieldPath` has no entry in `EBpirTypeKind`, `BpirTypeGrammar.cpp`, or `FindByPinCategory`. Affects UE 5.3+ MVVM nodes and any plugin using TFieldPath pins. Not observed in current PDS BPs but a silent break the moment such a node is added. Grammar extension `field_path<OwnerClass.PropertyName>` mirrors the existing `object<UClass>` form.
- `#2-bare-field-path-type` `IN-REVIEW` developer — Replaced the tagged FieldPath model with bare `field_path` plus `fieldpath` alias, mapped `PC_FieldPath` through type grammar/decompiler/default handling, and added make-pin-type/pin-default regression tests.
- `#3-verify-field-path-type` `DONE` tester — Verified: created `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_pc_fieldpath_absent`, ran `blueprint.compile_bpir` with `entry function VerifyFieldPath(field_path Field, array<field_path> Fields, fieldpath Legacy) {}` and got `compiled: true`, `status: "UpToDate"`, `errors: []`; `blueprint.decompile` returned `entry function VerifyFieldPath(field_path Field, array<field_path> Fields, field_path Legacy)`, proving `field_path`, `array<field_path>`, and legacy `fieldpath` route through compile/decompile. Temp asset deleted via `asset.delete path`.
