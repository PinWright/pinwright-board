---
id: B-niagara-custom-struct-converter-rawbytes
title: "Niagara registered custom-struct converters fall through to opaque rawBytes hex"
status: DONE
severity: Low
category: bug
tags: [niagara, parameter-types, raw-bytes, struct-converter]
---

# Niagara registered custom-struct converters still fall through to `_kind: rawBytes`

User-registered struct converters (registered with Niagara via the
`FNiagaraTypeDefinition` converter system — `GetStructConverter`,
`AddTypeToRegistry`, etc.) still emerge in `niagara_parameters.json` as opaque

```json
{ "_kind": "rawBytes", "sizeBytes": N, "hex": "..." }
```

This was deferred from the parent ticket
[B-niagara-bool-vec2-struct-rawbytes](B-niagara-bool-vec2-struct-rawbytes.md)
once the typed cases for bool / Vec2 / FNiagaraID / FNiagaraSpawnInfo /
half / half2 / half3 / half4 shipped. Custom-struct converters use a
different code path (`FNiagaraTypeDefinition::GetStructConverter` and the
LWC / SWC conversion-step machinery) that needs separate investigation.

## Scope

- `BuildParameterValueJson` in
  `Source/PinWright/Private/Handlers/Niagara/NiagaraDumpBuilder.cpp`
  — needs a final case before the rawBytes fallback that uses
  `CopyParameterData` and reflected `PropertyUtils` JSON export for
  registered script structs.
- Matching `set_parameter` writeback path in
  `NiagaraEditTypes.cpp::ValidateTypedValue` and
  `NiagaraEditHandler.cpp::ResolveNiagaraParameterType` /
  `SetTypedParameterValue` using reflected `PropertyUtils` JSON import and
  `SetParameterData`.

## Investigation hooks

- Walk a `FNiagaraTypeDefinition` whose underlying `UScriptStruct` was
  registered through Niagara's type registry. Confirm whether
  `GetStructConverter()` returns a non-null converter and whether it
  exposes typed property iteration.
- Reuse the existing reflected-property fallback (`PropertyUtils::ExportPropertyToJsonValue`
  pattern) once we can resolve the underlying script struct.

## History
- `#1-spinoff-from-bool-vec2-fix` `OPEN` reporter — Spun off from B-niagara-bool-vec2-struct-rawbytes after the typed-cases fix shipped. Custom-struct converter path uses different machinery (FNiagaraTypeDefinition::GetStructConverter) and was deferred.
- `#2-reflected-custom-struct-json` `IN-REVIEW` developer — Added reflected Niagara script-struct dump/writeback using parameter-store CopyParameterData/SetParameterData and PropertyUtils JSON helpers; rawBytes remains only for unsupported opaque types.
- `#3-returned-type-rejected` `OPEN` tester — Returned: duplicated `/App/App/FXE_Trail` to `/Game/App/UI/Test/NS_McpVerifyTemp_CustomStructRawbytes`, then `niagara.set_parameter` with `scope: user`, `name: User.McpVerifyConvertedVector`, `type: /Script/CoreUObject.Vector`, and vector fields returned `INVALID_PARAMETER_TYPE: Unsupported Niagara parameter type '/Script/CoreUObject.Vector'`. The reflected custom-struct writeback path from `#2` is not live for the same type shape used by the regression fixture; temp duplicate was deleted.
- `#4-vector-path-alias` `IN-REVIEW` developer — Added `/Script/CoreUObject.Vector` as a safe vec3 type alias in validation and mutation, plus a regression test covering the returned set_parameter shape; reflected registered script-struct dump/writeback remains covered separately.
- `#5-verify-vector-alias` `DONE` tester — Verified: duplicated `/App/App/FXE_Trail` to `/Game/App/UI/Test/NS_McpVerifyTemp_CustomStructRawbytes`, then `niagara.set_parameter` with `scope: user, name: User.McpVerifyConvertedVector, type: /Script/CoreUObject.Vector, value: {x:1,y:2,z:3}` now returns `PARAMETER_NOT_FOUND` (the parameter doesn't exist on this asset, an expected post-validation error) instead of the prior `INVALID_PARAMETER_TYPE: Unsupported Niagara parameter type '/Script/CoreUObject.Vector'`. The vec3 alias is live; temp duplicate deleted.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
