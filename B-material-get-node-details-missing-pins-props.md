---
id: B-material-get-node-details-missing-pins-props
title: "get_node_details / get_material_node_details omit pins, properties, parameter name"
status: OPEN
severity: High
category: bug
tags: [material, material-graph, material-authoring, get-node-details, inspection, asset-get-material-node-details]
encounters: 2
lastSeen: 2026-07-11T12:58:15.2039767+03:00
---

# Material node inspection returns a stub envelope

Both surfaces for inspecting a material expression return only the
node's class name, GUID, and editor position — none of the pin
connectivity, properties, or parameter metadata the wiki promises.

**`material.graph.get_node_details`** (`MaterialGraphHandler.cpp` ~L358):

```cpp
Result->SetStringField(TEXT("nodeType"), TargetExpr->GetClass()->GetName());
Result->SetStringField(TEXT("desc"), TargetExpr->Desc);
Result->SetNumberField(TEXT("x"), TargetExpr->MaterialExpressionEditorX);
Result->SetNumberField(TEXT("y"), TargetExpr->MaterialExpressionEditorY);
Ctx.SendSuccess(Result);
```

Wiki copy: *"Return detailed info (class, position, pin connectivity,
parameter name) for one node, or list every node in the graph when nodeId
is omitted."* Pin connectivity and parameter name are not in the payload.

**`material.authoring.get_material_node_details`** (`MaterialAuthoringHandler.cpp`
~L2438) is similar — `nodeId`, `nodeType`, `nodeName`, nothing else.

The bulk list mode (when `nodeId` is omitted) emits one entry per node
with only `{nodeId, nodeType, index, desc}` — no parameter name, no input
connections, no asset path for parameter / texture-sample nodes.

**Why it matters:**

- Round-trip authoring is impossible. After `add_expression` + `connect_nodes`,
  the caller cannot read back what the graph looks like — there's no way
  to verify wiring without falling back to `material.decompile_mgir` or
  `python.execute`.
- Surgical-edit workflows (find node N, inspect, decide whether to rewire)
  break at the inspection step.
- The asymmetry with `blueprint.graph.get_node_details` is jarring — that
  method returns full pin lists with types, default values, and current
  connections.

**Fix:**

Augment the response with:

1. **`inputs[]`**: walk `FExpressionInputIterator{Expr}`, emit
   `{name: GetInputName(Index).ToString(), connectedNodeId: Input->Expression
   ? Input->Expression->MaterialExpressionGuid.ToString() : "",
   outputIndex: Input->OutputIndex, hasMask: Input->Mask != 0, …}`.
   For function call inputs, also report the raw `FunctionInputs[i].Input.InputName`
   alongside the decorated `GetInputName` so callers can pick either.
2. **`outputs[]`**: `Expr->GetOutputs()` (UE 5.6 has `TArray<FExpressionOutput>`)
   — emit `{name, mask, …}` so multi-output nodes (BreakOutFloat3, MakeFloat3,
   function calls) can be addressed by name.
3. **`parameterName`**: if `Cast<UMaterialExpressionParameter>` succeeds,
   emit the name + `Group` + `SortPriority`.
4. **`properties{}`**: select reflected UPROPERTYs the caller can usefully
   set (mirror what `material.graph.add_expression`'s `properties` map
   accepts — `CoordinateIndex`, `UTiling`, `VTiling`, `ConstCoordinate`,
   `Speed`, `Code`, `OutputType`, `Description`, `SamplerType`, `Texture`
   path). At minimum, dump all non-default scalar/string/enum/object
   properties via the same reflection used elsewhere in the plugin.
5. **Texture sample**: emit `texturePath` (current `Texture->GetPathName()`)
   and `samplerType`.
6. **Pick one canonical handler.** The two methods do almost the same
   thing with different param shapes (`assetPath`+`nodeId` vs
   `materialPath`+`nodeId`). Deprecate one or have it forward to the
   other.

## Repro

1. Create material `M`, `add_scalar_parameter(name: "Roughness",
   defaultValue: 0.5, group: "Surface")` → returns nodeId `N`.
2. `material.graph.get_node_details(materialPath: M, nodeId: N)` →
   response has no `parameterName`, no `defaultValue`, no `group`.
3. Connect M to BaseColor via Main.
4. `get_node_details` on the same node still has no `outgoing` /
   `connections` field. Cannot tell that this node feeds BaseColor.

## History
- `#1-skeletal-inspection-payload` `OPEN` reporter — Material API audit: both `material.graph.get_node_details` and `material.authoring.get_material_node_details` return only class name, GUID, editor position, and (in the graph variant) `desc`. The wiki promises "pin connectivity, parameter name" but neither is in the payload. The bulk list mode is similarly stripped. Round-trip authoring (add → inspect → rewire) requires `material.decompile_mgir` as a workaround. Augment with `inputs[]` from `FExpressionInputIterator`, `outputs[]` from `GetOutputs()`, parameter name + Group + SortPriority for `UMaterialExpressionParameter` subclasses, and a reflected `properties{}` dump aligned with what `material.graph.add_expression` accepts. Consolidate the two near-duplicate methods.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Confirmed both handlers emit exactly the stub envelope described. `MaterialGraphHandler.cpp` L390-396 sets only `nodeType`/`desc`/`x`/`y` (plus `AddAssetVerification`); bulk-list branch L406-412 emits `nodeId`/`nodeType`/`index`/`desc`. `MaterialAuthoringHandler.cpp` L2464-2468 sets only `nodeId`/`nodeType`/`nodeName`. Wiki copy on the `get_node_details` registration (L359) literally says "pin connectivity, parameter name" — both absent, so this is a contract violation not a doc gap. Severity High justified: blocks round-trip authoring after every `add_expression` + `connect_nodes`. No duplicate board tickets. Implementation note for whoever picks this up: reusable infra already exists — `MGIR/MGIRDecompiler.cpp` L186-194 and `MGIR/MGIRExpressionUtils.h` L90 already walk `FExpressionInputIterator` + `GetInputName`, `MGIR/MGIRPinResolver.cpp` enumerates available input names, and `Utils/PropertyUtils.h` exposes `ExportPropertyToJsonValue` / `BuildSparsePropertyDiffJson` / `BuildClassPropertyJson` for the reflected properties dump. The fix should call these rather than re-rolling the walk/dump logic.
- `#3-shared-details-helper` `IN-REVIEW` developer — Added BuildExpressionDetailsJson(UMaterialExpression*) in MGIR/MGIRExpressionUtils.h emitting inputs[]/outputs[]/parameterName/group/sortPriority/properties/texturePath/samplerType; MaterialGraphHandler.cpp single-node branch (~L368) and MaterialAuthoringHandler.cpp get_material_node_details (~L2439) now share it. Bulk-list branch unchanged to keep responses compact. Regression test TestMaterialGetNodeDetails.cpp asserts parameterName/group/properties.DefaultValue/outputs on a Roughness scalar parameter.
- `#4-verify-details-helper` `DONE` tester — Verified: created /Game/McpVerify/M_McpVerifyTemp_BMaterialGetNodeDetails, added scalar parameter Roughness with defaultValue 0.5/group Surface/sortPriority 7, connected it to Main.Roughness, and confirmed both material.graph.get_node_details and material.authoring.get_material_node_details returned outputs[], parameterName Roughness, group Surface, sortPriority 7, and properties.DefaultValue 0.5; deleted the temp asset afterward.
- `#5-regression-asset-surface` `OPEN` reporter — Regression / incomplete fix: the #3 consolidation routed `material.graph.get_node_details` and `material.authoring.get_material_node_details` through `MGIRExpressionUtils::BuildExpressionDetailsJson`, but a THIRD, undeprecated inspection surface — `asset.get_material_node_details` (AssetMaterialHandler.cpp L928-1046) — was never converted and still hand-rolls the old stub envelope. This is the method an agent naturally reaches for alongside `asset.add_material_parameter` (`materialPath` + numeric `expressionIndex`). Replayed on the Tint VectorParameter of /Game/Materials/M_EnvProp_Master: `asset.get_material_node_details {expressionIndex:0}` returns only `name` = "MaterialExpressionVectorParameter_0" (the UObject name, itself misleading for a parameter node), `class`, `classPath`, `editorX`, `editorY`, `inputs:[]` — no parameterName, no outputs, no group/sortPriority, no properties. The fixed sibling `material.authoring.get_material_node_details {nodeId:"MaterialExpressionVectorParameter_0"}` on the SAME node returns parameterName "Tint", the full outputs[] mask set, group, sortPriority, and a reflected properties{} dump including ParameterName. Additionally, the asset.* type-specific block (L1007-1042) only covers Constant / Constant2Vector / Constant3Vector / Constant4Vector / TextureSample, so for Scalar/Vector/StaticSwitch PARAMETER nodes it also omits the DefaultValue (confirmed on idx1 ScalarParameter Roughness and idx2 StaticSwitchParameter UseDetail — both returned no value field). Fix: route `asset.get_material_node_details` through the same `BuildExpressionDetailsJson` helper (or forward it to material.authoring), and extend the #4 regression test to assert parameterName on the asset.* surface too.
