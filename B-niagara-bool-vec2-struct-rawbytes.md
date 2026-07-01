---
id: B-niagara-bool-vec2-struct-rawbytes
title: "NiagaraBool parameters still dump as rawBytes because bool type matching uses strict FNiagaraTypeDefinition equality"
status: DONE
severity: Medium
category: bug
tags: [niagara, parameter-types, raw-bytes, type-system]
---

# Niagara parameter values fall through to opaque `_kind: rawBytes`

`BuildParameterValueJson` in the Niagara dump pipeline handles a
specific list of typed parameter shapes (float, int32, FVector3f,
FVector4f, FQuat4f, FLinearColor, IsDataInterface, IsUObject) and
falls everything else through to:

```json
{ "_kind": "rawBytes", "sizeBytes": N, "hex": "deadbeef..." }
```

This is **safe** (no `?` / `unknown` markers) but **opaque** for
common types:

| Type | Falls through? | Bytes |
|------|---------------|------:|
| `bool` | yes | 1 |
| `FVector2f` (Vec2) | yes | 8 |
| `FNiagaraID` | yes | 8 |
| `FNiagaraSpawnInfo` | yes | 16 |
| `half` / `half2` / `half3` / `half4` | yes | 2/4/6/8 |
| Custom struct (registered converter) | yes | varies |

Bool and Vec2 are common enough that opaque hex is clearly wrong
— `bool` should be `true`/`false`, `Vec2` should be `{x, y}`.

## Why it matters

`niagara.set_parameter` cannot write any of these types in their
typed form. Callers must:
- Hex-encode a single byte for bool (`00` or `01`).
- Construct an 8-byte hex blob for Vec2, knowing the in-memory
  layout.
- Re-derive struct layout for FNiagaraID and FNiagaraSpawnInfo.

This is unusable in practice. Any LLM-driven workflow that wants
to set a bool parameter (extremely common — emitter `bEnabled`,
module gates, sampling toggles) is blocked.

## Fix

The original typed bool handling was present, but it matched bool with
strict `FNiagaraTypeDefinition` equality. `/App/App/FXE_Trail` exposes
`NiagaraBool` as a struct-backed type with the same base definition as
`FNiagaraTypeDefinition::GetBoolDef()` and 4-byte storage, so strict
equality bypasses the bool branch and falls through to `_kind: rawBytes`.

Match bool-family types with
`Type.IsSameBaseDefinition(FNiagaraTypeDefinition::GetBoolDef())` in
the dump/model serializers. Keep the byte-OR bool decode so both the
4-byte `FNiagaraBool` storage and any 1-byte parameter-store
representation serialize to JSON booleans. The edit path should accept
`NiagaraBool` as a bool spelling and allow existing-parameter type
compatibility through the same base-definition relationship before
rejecting `PARAMETER_TYPE_MISMATCH`.

## Repro

Inspect any `niagara_parameters.json` with a bool or Vec2
parameter. Sample:
- Bool emerges as `{ "_kind": "rawBytes", "sizeBytes": 1, "hex": "01" }`.
- Vec2 emerges as `{ "_kind": "rawBytes", "sizeBytes": 8, "hex": "..." }`.

## History
- `#1-initial-spec` `OPEN` reporter — Niagara type-system audit confirmed `BuildParameterValueJson` covers float / int32 / Vec3 / Vec4 / Quat / LinearColor / DI / UObject and falls through to `_kind: rawBytes` for everything else. Bool and Vec2 are common enough to make the rawBytes fallback unusable. Add typed cases for bool, Vec2, FNiagaraID, FNiagaraSpawnInfo, half variants, and registered-struct converters; matching write paths in `set_parameter`.
- `#2-typed-cases-implemented` `IN-REVIEW` developer — Implemented typed cases for bool, Vec2, FNiagaraID, FNiagaraSpawnInfo, half, half2, half3, half4 in BuildParameterValueJson + ValidateTypedValue + ResolveNiagaraParameterType + SetTypedParameterValue. Hoisted JsonVec2, JsonHalf, ExtractVector2fField, ExtractFloat16Field into Utils/JsonUtils. Custom-struct converters spun off to B-niagara-custom-struct-converter-rawbytes (different machinery). Regression test FNiagaraDumpBuilderTypedParameterCoverageTest.
- `#3-review-iteration-1` `IN-REVIEW` developer — Fix iteration 1: added WHY comment explaining byte-OR bool dump; refactored ResolveNiagaraParameterType + SetTypedParameterValue to share TryParseNiagaraTypeName lookup (eliminates duplicated string-compare chains and dead case-pair duplicates); composed JsonUtils::JsonHalf into half2/half3/half4 dump branches (or added JsonHalfVecN helpers).
- `#4-returned-bool-still-rawbytes` `OPEN` tester — Returned: fresh `asset.dump` on `/App/App/FXE_Trail` writes typed `Vector2f` values as `{x,y}` but still writes `NiagaraBool` values as `_kind:"rawBytes"` (`hex:"00000000"`, `sizeBytes:4`) in `niagara_parameters.json`. Test: `mcp__editor_automation__.call path="asset.dump" args={"assetPath":"/App/App/FXE_Trail"}` then inspected `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/FXE_Trail/niagara_parameters.json`.
- `#5-bool-base-type-match` `IN-REVIEW` developer — Changed Niagara bool dump/model serialization and set_parameter type compatibility to match bool by base Niagara type definition instead of strict type equality; added regression coverage for the returned NiagaraBool rawBytes shape.
- `#6-verify-fix` `DONE` tester — Verified: fresh `asset.dump` on `/App/App/FXE_Trail` writes both NiagaraBool entries (lines 738, 1000) as typed `{ name: "NiagaraBool", sizeBytes: 4, struct: "/Script/Niagara.NiagaraBool" }` with `"value": false`. Zero `rawBytes` occurrences remain in `niagara_parameters.json`.
