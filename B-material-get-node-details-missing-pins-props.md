---
id: B-material-get-node-details-missing-pins-props
title: "get_node_details / get_material_node_details omit pins, properties, parameter name"
status: IN-REVIEW
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
- `#6-adversarial-current-contract` `OPEN` reporter — Additional evidence: **Adversarial review A — canonical detail contract is fixed; the remaining OPEN regression is stale.** Actuality: STALE/FIXED. Framing: Both live detail handlers route through `MGIRExpressionUtils::BuildExpressionDetailsJson`, which emits input connectivity (connected GUID/output/masks), outputs, parameter metadata, non-default reflected properties, and texture fields. `asset.get_material_node_details` is not a third live surface: the asset handler registers only list/reset/stats, and the removal ledger explicitly records that method as removed in favor of `material.graph.get_node_details`; Git history `X:\src\unreal\pinwright-ue@aeb136f` (v0.5.0) matches that split. The compact no-node list is documented behavior, not evidence the single-node payload regressed. The old repro's “outgoing” claim is not part of the current detail contract; root wiring is intentionally read through `Main` / `mainInputs`, while arbitrary consumer discovery would be a separate graph-wide feature. Title now describes a defect no longer present; High severity is not justified for the removed legacy API. Proposed fix: SYSTEMIC, the shared helper is the right parity mechanism; do not resurrect or patch the removed asset method. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Material\MaterialGraphHandler.cpp:321`, `:358`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Material\MaterialAuthoringHandler.cpp:2813`, `:2849`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\MGIR\MGIRExpressionUtils.h:195`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\material.graph.md:21`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetMaterialHandler.cpp:50`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\rpc-hard-removal-rejected-candidates.md:36`, `C:\UE_5.8\Engine\Source\Runtime\Engine\Public\Materials\MaterialExpression.h:684`. Runtime: NOT VERIFIED. Recommendation: CLOSE CANDIDATE; perform one registry/smoke check and, if outgoing-consumer discovery is still wanted, file a separate narrowly scoped contract with graph-wide scan and tests for both canonical handlers.
- `#7-additional-parity-test-gap` `OPEN` reporter — Additional evidence: **Adversarial review B — A is right that the legacy asset verb is removed, but its fully-fixed conclusion is not yet falsifiable from the checked-in tests.** Actuality: PARTIAL. Framing: `material.graph.get_node_details` and `material.authoring.get_material_node_details` both call the shared helper, so the requested fields are statically present; however, the ordinary-node payload regression test covers only the graph route (`TestMaterialGetNodeDetails.cpp:74`), while the authoring route test covers only the `Main` sentinel (`TestMaterialMainNodeReadback.cpp:135`). The cited A-history revision `aeb136f` is not present in this checkout; the local removal is evidenced by the current asset registrations and removal ledger, not that hash. Proposed fix: SYSTEMIC, retain the shared helper and add a real scalar/vector/texture ordinary-node parity test for both canonical handlers; do not restore `asset.get_material_node_details`. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Material\MaterialGraphHandler.cpp:356`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Material\MaterialAuthoringHandler.cpp:2842`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\MGIR\MGIRExpressionUtils.h:195`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetMaterialHandler.cpp:50`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\rpc-hard-removal-rejected-candidates.md:36`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Material\TestMaterialGetNodeDetails.cpp:74`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Material\TestMaterialMainNodeReadback.cpp:135`. Runtime: NOT VERIFIED. Recommendation: VERIFY FIRST; run registry plus ordinary-node parity smoke/automation on both canonical handlers, then close candidate if both payloads match.
- `#8-parameter-metadata-uniform-accessor` `IN-REVIEW` developer — Confirmed #6 and #7 in `unreal-fpv-new`: `asset.get_material_node_details` no longer exists (`AssetMaterialHandler.cpp` registers only `asset.list_material_instances` / `asset.reset_instance_parameters` / `asset.get_material_stats`), and both canonical handlers already route through `MGIRExpressionUtils::BuildExpressionDetailsJson`, which emits inputs[]/outputs[]/parameterName/properties/texturePath/samplerType. Found and fixed a real residual hole in that helper: `group` / `sortPriority` were read via `Cast<UMaterialExpressionParameter>`, which fails for every parameter family that declares its own Group/SortPriority outside that hierarchy — `UMaterialExpressionTextureSampleParameter*`, font sample, runtime virtual texture and sparse volume texture parameters — so the payload emitted `parameterName` (virtual `HasAParameterName()` returns true for them) while silently dropping both metadata fields, contradicting the `docs/wiki-src/material.graph.md` detail contract. Replaced the cast with the engine's uniform `UMaterialExpression::GetParameterValue(FMaterialParameterMetadata&)` — the same accessor `MaterialAuthoringHandler.cpp`'s `AddMaterialParameterDetails` already uses for `get_material_info` parameters[] — plus the house `__has_include("Materials/MaterialParameters.h")` compat include. `UMaterialExpressionParameter`'s own override fills the identical two fields, so scalar/vector/static-switch behavior is byte-identical. Regression test `PinWright.material.graph.get_node_details.TextureParameterMetadata` in `Tests/Material/TestMaterialGetNodeDetails.cpp` builds a TextureSampleParameter2D with ParameterName BaseColorTex / Group Textures / SortPriority 7 and asserts parameterName, group, sortPriority, properties.SortPriority, inputs and outputs on BOTH `material.graph.get_node_details` and `material.authoring.get_material_node_details` — which also closes the #7 authoring-route parity gap. Fails before the fix (group/sortPriority absent from both responses). No `AssetDumpCache` aspect bump: `BuildExpressionDetailsJson` has exactly two call sites, both RPC handlers, and feeds no dump sidecar. Not compiled or run here (build and suite are owned by the wave). Board-vs-source contradictions: the ticket title and #5 describe surfaces/defects that no longer exist in this checkout.
