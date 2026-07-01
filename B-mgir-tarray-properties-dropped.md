---
id: B-mgir-tarray-properties-dropped
title: "MGIR decompile silently drops TArray UPROPERTYs — Custom HLSL multi-output, defines, includes, dynamic-param channel names lost"
status: DONE
severity: High
category: bug
tags: [mgir, decompiler, tarray, custom-hlsl, data-loss]
---

# `IsSafeReflectedProperty` rejects FArrayProperty — silent data loss

`MGIRDecompiler.cpp:313-358` `IsSafeReflectedProperty()` checks
`IsA<>` for: Bool, Numeric, Enum, Byte, Str, Name, Text, Object,
Class, SoftObject, SoftClass, Struct. **`FArrayProperty` is not
in the list** — every `TArray<>` UPROPERTY on every
UMaterialExpression is silently excluded from the reflected
property dump.

## Specifically lost data

| UPROPERTY | Owner | Impact |
|-----------|-------|--------|
| `AdditionalOutputs` (`TArray<FCustomOutput>`) | `UMaterialExpressionCustom` | Multi-output pin declarations dropped → node decompiles as single-output, downstream `%ref[1]`, `%ref[2]` connections broken |
| `AdditionalDefines` (`TArray<FCustomDefine>`) | `UMaterialExpressionCustom` | All HLSL `#define` macros dropped |
| `IncludeFilePaths` (`TArray<FString>`) | `UMaterialExpressionCustom` | Custom shader include files dropped |
| `ParamNames` (`TArray<FString>`) | `UMaterialExpressionDynamicParameter` | All four channel names default back to `"Param0"`–`"Param3"` after compile |
| Various other engine expressions | (long tail) | Anything else with `TArray<>` UPROPERTY |

## Compounding compiler gap

Even if the decompiler emitted these arrays, `ApplyJsonValueToProperty`
in `PropertyUtils.cpp` covers only String / Name / Text / Bool /
Float / Double / Int / Byte / Object **inner** types in array
handling. `FStructProperty` inner is not handled — returns
`"Unsupported array inner property type"`. So
`Custom.AdditionalOutputs` (array of `FCustomOutput` structs)
would also fail on compile even after the decompile fix.

## Fix

Two-sided:

1. **Decompiler:** add `FArrayProperty` to `IsSafeReflectedProperty`,
   and emit array values via the existing reflected-property pipeline.
   For struct-inner arrays, recurse into each element via the
   sparse-property-diff logic already used for struct properties.
2. **Compiler:** extend `ApplyJsonValueToProperty`'s array-inner
   handling to support `FStructProperty` inner. Allocate the array,
   iterate elements, recurse into struct property setting.

## Why high severity

Custom HLSL nodes are a primary extensibility point for materials.
Any material with non-trivial Custom node usage (multi-output,
defines, includes) is silently corrupted on round-trip. Cache shows
97+ materials with Custom HLSL nodes.

## Repro

`asset.dump` then `mgir.txt` of a material with a Custom node that
declares additional outputs (e.g. via the editor's "+" button under
Output Type). Compare `mgir.txt` against the asset's actual
properties — the `AdditionalOutputs` list is absent.

## History
- `#1-initial-spec` `OPEN` reporter — MGIR coverage parity audit confirmed `IsSafeReflectedProperty` (MGIRDecompiler.cpp:313-358) rejects `FArrayProperty`. Drops Custom HLSL multi-output declarations, defines, includes, and DynamicParameter channel names. Compounding gap on the compiler side: `ApplyJsonValueToProperty` doesn't handle struct-inner arrays. Both must be fixed to round-trip Custom nodes.
- `#2-array-property-roundtrip` `IN-REVIEW` developer — Added FArrayProperty to MGIRDecompiler IsSafeReflectedProperty whitelist + struct-inner array branch in PropertyUtils ApplyJsonValueToProperty. Regression test FApplyJsonValueToProperty_StructArrayRoundTrip in TestMGIRDecompilerArrayProperties.cpp.
- `#3-verify-array-roundtrip` `DONE` tester — Verified: created `/Game/McpVerify/M_McpVerifyTemp_B_mgir_tarray_properties_dropped`, used `material.graph.add_expression` with Custom `AdditionalOutputs`, `AdditionalDefines`, and `IncludeFilePaths`, then `material.decompile_mgir` returned all three array properties including `OutputName="Alpha"`, `OutputName="RGB",OutputType=CMOT_Float3`, `DefineName="MCP_VERIFY_DEFINE"`, and `/Project/McpVerify.ush`; temp asset deleted afterward with `asset.delete path`.
