---
id: E-bpir-type-prefix-tolerance
title: "BPIR type-reference slots reject canonical U/A/F/I prefixes"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, ergonomics, symbol-resolution]
---

# BPIR type-reference slots reject canonical U/A/F/I prefixes

BPIR's symbol resolver strips the leading `U`/`A`/`F`/`I` prefix from class
names inside angle-bracket type slots (e.g. `subsystem<...>`,
`cast<...>`, `make<...>`). Passing the canonical C++ identifier as it
appears in headers, IDE auto-complete, and doc references fails to
resolve.

**Repro:** `blueprint.compile_bpir` with
`%n0 = subsystem<UReplaySaveWorldSubsystem>()` returns
`COMPILE_FAILED: Unresolved subsystem class: UReplaySaveWorldSubsystem`.
Stripping the `U` (`subsystem<ReplaySaveWorldSubsystem>`) compiles.

Authors who type the canonical class name lose one compile attempt per
new symbol. The behaviour is undocumented and inconsistent with the
unified class-name resolver from `E-class-name-format-inconsistency`,
which accepts U/A-prefixed names on tool parameters.

**Scope:** Applies to every BPIR slot that names a UClass/UScriptStruct
by short name — confirmed for `subsystem<...>`; likely also affects
`cast<UFoo>`, `make<FBar>`, interface refs, and any other angle-bracket
type reference. Fix should be applied uniformly at the resolver level,
not per-slot.

**Workaround:** Drop the leading `U`/`A`/`F`/`I` prefix.

**Fix:** Route BPIR type-name resolution through (or in the spirit of)
`ResolveUClass` so both `UReplaySaveWorldSubsystem` and
`ReplaySaveWorldSubsystem` resolve. Update
`bpir-language-reference.md` to state which forms are accepted.

## History
- `#1-subsystem-u-prefix-rejected` `OPEN` reporter — Hit during a session compiling `subsystem<UReplaySaveWorldSubsystem>()`; resolver returned `Unresolved subsystem class`. Stripping `U` compiled. Likely affects other angle-bracket type slots (`cast<>`, `make<>`, interface refs) — fix should cover all of them at the resolver layer.
- `#2-route-slots-through-unified-resolvers` `IN-REVIEW` developer — Subsystem/MakeStruct/BreakStruct opcode arms in `BpirCompiler.cpp` (the three slots that bypassed the unified resolvers) now route through `ResolveUClass` / `ResolveUScriptStruct`, which already strip U/A/F prefixes plus other ergonomics. Replaced the manual `FindFirstObjectSafe` + prepend-only retry pairs. Regression test `TestBpirTypePrefixTolerance.cpp` added with three cases covering `subsystem<UGameInstanceSubsystem>`, `make<FVector>`, `break<FVector>`. `Cast` and `SwitchEnum` slots were already routing through the unified resolvers and required no change.
- `#3-verify-prefix-tolerance` `DONE` tester — Verified: created `/Game/App/UI/Test/W_McpVerifyTemp_E_bpir_type_prefix_tolerance`, ran `blueprint.compile_bpir` with `subsystem<UGameInstanceSubsystem>()`, `make<FVector>(...)`, and `break<FVector>($v)`, observed `success: true`, `compiled: true`, `errors: []`, `nodeCount: 5`, then deleted the temp asset.
