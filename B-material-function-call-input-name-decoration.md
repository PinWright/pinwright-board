---
id: B-material-function-call-input-name-decoration
title: "Material function call inputs unwireable by raw name — GetInputName decoration mismatch"
status: DONE
severity: High
category: bug
tags: [material, material-authoring, material-function, connect-nodes, pin-resolution]
---

# `connect_nodes` can't wire material function call inputs by their declared name

The wiki page for `material.authoring` documents the limitation:
*"Material function call input pins are not reliably discoverable through
MCP, so they cannot be wired by name in practice."*

Root cause confirmed in engine source. `UMaterialExpressionMaterialFunctionCall::GetInputName`
(in `Engine/Source/Runtime/Engine/Private/Materials/MaterialExpressions.cpp`
~L17353) returns the input name **with a type suffix** appended:

```cpp
return *FString::Printf(TEXT("%s (%s)"),
    *FunctionInputs[InputIndex].Input.InputName.ToString(),
    GetInputTypeName(FunctionInputs[InputIndex].ExpressionInput->InputType));
```

So a function with input `Roughness` of type `Float1` reports its input
name as `"Roughness (S)"`, with input `Color` of type `Float3` reports
`"Color (V3)"`, etc.

The plugin's `FindExpressionInputByName`
(`MaterialAuthoringHandler.cpp` L132, also via `FMaterialExpressionFactory`)
does:

```cpp
for (FExpressionInputIterator It{ Expr }; It; ++It)
{
    if (Expr->GetInputName(It.Index).ToString()
        .Equals(InputName, ESearchCase::IgnoreCase))
    {
        return It.Input;
    }
}
```

`Equals(InputName, IgnoreCase)` requires the caller to pass `"Roughness (S)"`
verbatim. Callers have no way to know the type suffix without inspecting
the function (which is one of the things `B-material-get-node-details-missing-pins-props`
fixes for general inspection but doesn't address for function call sites
specifically).

The earlier branch in the same helper:

```cpp
if (FProperty* Prop = Expr->GetClass()->FindPropertyByName(FName(*InputName)))
{
    if (FStructProperty* StructProp = CastField<FStructProperty>(Prop))
        if (StructProp->Struct->GetFName() == FName(TEXT("ExpressionInput")))
            return StructProp->ContainerPtrToValuePtr<FExpressionInput>(Expr);
}
```

doesn't help here — function call inputs live in `FunctionInputs[i].Input.InputName`,
not as named `FExpressionInput` UPROPERTYs on the class, so reflection
finds nothing and falls through to the failing decorated-name path.

This is the most user-visible blocker for material-function-driven
authoring. Workaround the wiki suggests — "use direct material nodes or
Custom HLSL when a function such as CustomRotator needs inputs" — defeats
the purpose of `use_material_function`.

**Fix:**

In `FindExpressionInputByName`, add a function-call special case
**before** the iterator loop:

```cpp
if (UMaterialExpressionMaterialFunctionCall* FuncCall =
    Cast<UMaterialExpressionMaterialFunctionCall>(Expr))
{
    for (int32 i = 0; i < FuncCall->FunctionInputs.Num(); ++i)
    {
        const FFunctionExpressionInput& FuncInput = FuncCall->FunctionInputs[i];
        if (FuncInput.Input.InputName.ToString()
            .Equals(InputName, ESearchCase::IgnoreCase))
        {
            return const_cast<FExpressionInput*>(&FuncInput.Input);
        }
    }
}
```

This resolves by the **raw** `InputName` (the function's declared input
name), not by the decorated `"Name (Type)"` form. The iterator loop can
remain as a fallback for the decorated form, so existing callers that
already pass `"Roughness (S)"` keep working.

Apply the same fix to `material.graph.connect_nodes`'s pin resolver
(`FMaterialExpressionFactory::FindExpressionInputByName`) — the file
`Material/MaterialExpressionFactory.cpp` has the equivalent loop.

Similarly add a function-output special case so `sourcePin` resolution
(once `F-material-connect-source-pin-output-index` lands) can pick a
named function output without needing the type suffix.

## Repro

1. Create material function `MF` with input `Speed` (Float1).
2. Create material `M`, `use_material_function(MF)` → call node `C`.
3. Add scalar parameter `S` → node `N`.
4. `material.authoring.connect_nodes(assetPath: M, sourceNodeId: N,
   targetNodeId: C, inputName: "Speed")` → `PIN_NOT_FOUND: Input pin
   'Speed' not found.`
5. Workaround that currently succeeds: `inputName: "Speed (S)"`.

## History
- `#1-decorated-input-name-mismatch` `OPEN` reporter — Material API audit confirmed the documented "material function call inputs unwireable" limitation has a single concrete cause: `UMaterialExpressionMaterialFunctionCall::GetInputName` returns `"InputName (TypeSuffix)"` (engine `MaterialExpressions.cpp` L17353), but `FindExpressionInputByName` and `FMaterialExpressionFactory::FindExpressionInputByName` both compare with `Equals(InputName, IgnoreCase)` against the decorated form. Callers have no path to discover the type suffix. Fix is a function-call special case that walks `FunctionInputs[].Input.InputName` directly before falling back to the decorated-name iterator loop.
- `#2-reviewed-and-confirmed` `OPEN` tester — Verified against UE 5.6 source: `UMaterialExpressionMaterialFunctionCall::GetInputName` at `MaterialExpressions.cpp` L17363 unconditionally calls `GetInputNameWithType(InputIndex, true)` (L17347-17361), producing the `"Name (Type)"` decoration. Plugin code matches the ticket: `MaterialAuthoringHandler.cpp` L159 and `MaterialExpressionFactory.cpp` L80 both compare via `Expr->GetInputName(It.Index).ToString().Equals(InputName, IgnoreCase)`. No duplicate board entry. High severity justified — this is the single root cause behind the documented wiki limitation, not a corner case. Minor refinement on the proposed fix: the engine exposes `ENGINE_API FName GetInputNameWithType(int32, bool bWithType)` (header L141); calling `FuncCall->GetInputNameWithType(It.Index, false)` is a one-line alternative to manually indexing `FunctionInputs[]` and yields the raw `InputName` directly. Either approach works; the ticket's `FunctionInputs[].Input.InputName` walk is more explicit and self-contained.
- `#3-raw-name-comparison-added` `IN-REVIEW` developer — Added GetInputNameWithType(It.Index, false) comparison alongside existing decorated GetInputName comparison in both FindExpressionInputByName implementations: MaterialAuthoringHandler.cpp (~L134) and Material/MaterialExpressionFactory.cpp (~L80). Decorated form still works as a fallback. Regression test TestMaterialFunctionCallInputNameResolution.cpp covers raw-name resolution on a MaterialFunctionCall with FunctionInputs[0].Input.InputName="Speed".
- `#4-verify-raw-input-name` `DONE` tester — Verified: created `/Game/McpVerify/B_material_function_call_input_name_decoration/MF_McpVerify_RawInputName` with function input `Speed`, inserted it into `/Game/McpVerify/B_material_function_call_input_name_decoration/M_McpVerify_RawInputName`, and `material.authoring.connect_nodes` returned `Nodes connected.` for raw `inputName: "Speed"`; cleaned up the temp folder afterward.
