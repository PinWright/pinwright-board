---
id: E-bpir-struct-literal-formatter-duplicated
title: "FVector / FRotator / FLinearColor literal formatting duplicated across CodePinResolver and BpirValueResolver; should defer to UScriptStruct::ImportText"
status: DONE
severity: Medium
category: ergonomic
tags: [bpir, resolver, struct-literal, duplication]
---

# Struct literal formatter is duplicated and hardcoded

`SetPinDefaultValue` in `Compiler/CodePinResolver.cpp:111-161` and
`GetLiteralText` in `Compiler/BpirValueResolver.cpp:511-538` both
contain dedicated formatters for `FVector`, `FRotator`, and
`FLinearColor`:

- `CodePinResolver.cpp:111-127` — `FLinearColor(r,g,b,a)` →
  `(R=,G=,B=,A=)`
- `CodePinResolver.cpp:129-144` — `FVector(x,y,z)` → `(X=,Y=,Z=)`
- `CodePinResolver.cpp:146-161` — `FRotator(p,y,r)` → `(Pitch=,Yaw=,Roll=)`
- `BpirValueResolver.cpp:511-515` — `FVector` parse + format
- `BpirValueResolver.cpp:517-519` — `FRotator` parse + format
- `BpirValueResolver.cpp:522-538` — `FLinearColor` parse + format
- `BpirValueResolver.cpp:542-550` — generic `F<Name>(...)` strip
  fallback (fragile for structs with non-positional members)

Two problems:

1. **Duplication.** The same three structs are formatted in two
   places. Adding support for a fourth (e.g. `FVector2D`,
   `FIntPoint`, `FBox`) requires touching both files and keeping
   them in sync.
2. **Latent positional-CSV bug.** The `BpirValueResolver::GetLiteralText`
   FVector/FRotator branches strip the wrapper and return the raw
   CSV (`"1.5,-2.25,3.75"`), which K2 pin defaults cannot bind to
   `(X=,Y=,Z=)` keys. Pins silently zero on round-trip.
3. **Hardcoded set.** Every UE struct with a `WithImportText` trait
   already supports `UScriptStruct::ImportText` /
   `UScriptStruct::ExportText`, which is what the engine itself
   uses to round-trip pin defaults. The hand-rolled formatters
   reimplement a subset of that.

## Fix (implemented)

Centralized K2-validator-grammar registry in `BpirStructLiteralUtils::TryFormatPositionalStructLiteralAsPinText`,
emitting the exact form each engine validator accepts: FVector → `(X=,Y=,Z=)`,
FRotator → positional CSV `p,y,r`, FLinearColor → `(R=,G=,B=,A=)`. Prior
ImportText/ExportText round-trip approach failed for FRotator because UE's
ExportText emits `(Pitch=,Yaw=,Roll=)`, which K2's `IsStringValidRotator → IsStringValidVector`
validator rejects (it accepts only `X=Y=Z=` keys or positional CSV). Replaces
the FVector/FRotator/FLinearColor branches plus fragile `F<Name>(...)` strip
fallback in `BpirValueResolver.cpp`. The duplicate branches in
`CodePinResolver.cpp:111-161` were dead code (call-graph audit: every
production call site pre-runs `GetLiteralText` first) and were deleted, not
unified.

Genuinely-needs-special-case (do **not** route through ImportText):
- `PC_Text` pin (`CodePinResolver.cpp:67-88`) — FText carries
  namespace/key metadata that ImportText handles inconsistently.
- Numeric pin categories (`CodePinResolver.cpp:241-258`) — code
  comment notes `TrySetDefaultValue` mishandles numeric strings;
  direct assignment is intentionally chosen.

## History
- `#1-initial-spec` `OPEN` reporter — Audit of `CodePinResolver.cpp` and `BpirValueResolver.cpp` found FVector/FRotator/FLinearColor formatting duplicated across both files plus a fragile generic `F<Name>(...)` strip fallback. UE's `UScriptStruct::ImportText`/`ExportText` already does this generically.
- `#2-helper-and-dead-code-deletion` `IN-REVIEW` developer — Added `BpirStructLiteralUtils.h/.cpp` namespace helper using `ImportText`/`ExportText`. Replaced FVector/FRotator/FLinearColor branches + strip fallback in `BpirValueResolver.cpp`. Deleted dead branches in `CodePinResolver.cpp:111-161`. Test: `EditorAutomationRpcGateway.bpir.struct_literal.{Vector,Rotator,LinearColor}`.
- `#3-rotator-still-zeros` `OPEN` tester — Returned: FVector and FLinearColor round-trip correctly through compile_bpir+decompile (`FVector(11.500000,22.500000,33.500000)`, `FLinearColor(0.250000,0.500000,0.750000,1.000000)`), but FRotator still silently zeros — `FRotator(11.5, 22.5, 33.5)` on a `K2_SetActorRotation.NewRotation` pin decompiles back as `0, 0, 0`. Test: created `/Game/App/UI/Test/BP_McpVerifyTemp_StructLiteral` (Actor parent), compiled `entry function TestRot() { call SetActorRotation(NewRotation: FRotator(11.5, 22.5, 33.5)) @(300, 0) }`, then `blueprint.decompile` showed the zeroed pin. Likely root cause is the one the body called out and rejected: FRotator's ExportText emits `(P=,Y=,R=)` short keys, which K2 pin defaults parse as `Pitch=Yaw=Roll=0`. ImportText+ExportText round-trip is not equivalent to `(Pitch=,Yaw=,Roll=)` for this struct.
- `#4-k2-grammar-registry` `IN-REVIEW` developer — Reframed helper from ImportText/ExportText round-trip into a K2-validator-grammar registry: FRotator emits positional CSV (the form `IsStringValidVector` accepts as primary path), FVector keeps `(X=,Y=,Z=)`, FLinearColor keeps `(R=,G=,B=,A=)`. Test: `EditorAutomationRpcGateway.bpir.struct_literal.RotatorPinDefault`. Counterfactual: revert the FRotator branch to `(Pitch=,Yaw=,Roll=)` and the test fails because the resulting pin text contains `Pitch=`.
- `#5-skip-gateway-down` `SKIP` tester — RPC gateway unreachable: port 19880 closed (Test-NetConnection fail; MCP `call` returns `fetch failed`). UnrealEditor process running is a DebugGame build that doesn't expose the gateway, so compile_bpir + decompile round-trip on a temp BP couldn't be exercised. Re-verify after launching a Development-target editor with the plugin loaded.
- `#6-verify-rotator-roundtrip` `DONE` tester — Verified: created `/Game/App/UI/Test/BP_McpVerifyTemp_StructLiteralRot` (Actor parent), compiled `entry function TestRot() { call SetActorRotation(NewRotation: FRotator(11.5, 22.5, 33.5)) }` then `blueprint.decompile` returned `call K2_SetActorRotation(NewRotation: 11.5,22.5,33.5, bTeleportPhysics: false)` — non-zero round-trip confirms the K2-validator positional CSV form is being written to the pin (#3's failure case decompiled back as `0, 0, 0`). Cross-checked FVector with same flow: decompiles as `FVector(11.5,22.5,33.5)`. Temp BP cleaned up.
