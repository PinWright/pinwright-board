---
id: F-material-instance-overrides-incomplete
title: "Material instance API can't clear overrides, set static switches, reassign parent, or query inherited values"
status: IN-REVIEW
severity: High
category: feature
tags: [material, material-authoring, material-instance, parameters, coverage-gap, regression, static-switch]
---

# Material instance API covers only the override-set path

`material.authoring.create_material_instance` plus three setters
(`set_scalar_parameter_value`, `set_vector_parameter_value`,
`set_texture_parameter_value`) is the entire material-instance surface.
The missing operations leave several common workflows blocked.

## Gaps

**1. No way to clear a single override.**

`UMaterialInstanceConstant::ClearParameterValueEditorOnly(FMaterialParameterInfo)`
exists in UE 5.6 and lets you remove a specific override (returning the
slot to inheriting from the parent). There is no RPC for it. Workaround
is to delete the entire instance and recreate, which loses any other
overrides on the same instance.

**2. No static-switch parameter override.**

The `add_static_switch_parameter` RPC adds a static switch expression to
a **parent material**. On the **instance** side, you can't override the
boolean — `UMaterialInstanceConstant` stores static parameter values in
`FStaticParameterSet StaticParameters` via `SetStaticSwitchParameterValueEditorOnly`,
and there is no corresponding RPC. This blocks an extremely common
authoring pattern (binary feature toggles on per-instance basis: "is this
prop's material the rainy variant?").

**3. No parent-material reassignment.**

`Instance->SetParentEditorOnly(NewParent)` is the supported API for
swapping a material instance's parent. No RPC exposes it. Workaround is
to delete and recreate, which loses overrides.

**4. No way to read instance values back.**

`get_material_info` only accepts `UMaterial` — `LoadObject<UMaterial>` on
a `UMaterialInstanceConstant` returns nullptr, and the handler then
returns `UNSUPPORTED_ASSET_CLASS` (per the explicit class check at
`MaterialAuthoringHandler.cpp` L2143–2152). There is no `get_material_instance_info`
that returns:

- `parent` (asset path)
- `overrides`: `{ scalar: {Name: Value}, vector: {Name: {R,G,B,A}},
  texture: {Name: AssetPath}, staticSwitch: {Name: Bool} }`
- `inherited`: same shape but values from the parent (i.e. what
  `GetScalarParameterValue(FMaterialParameterInfo)` returns when not
  overridden)

Without this, a caller cannot verify that `set_scalar_parameter_value`
actually took effect (the call returns success unconditionally — see the
authoring handler — but doesn't echo back the new value).

**5. No batch parameter setter.**

`SetScalarParameterValueEditorOnly` triggers a per-call
`PostEditChangeProperty` / shader recompile. A batch path
`set_material_instance_parameters(assetPath, scalar: {...}, vector: {...},
texture: {...}, staticSwitch: {...})` would amortize the cost.

**6. `material.authoring.add_vector_parameter` and `add_scalar_parameter`
lack `SortPriority`.** The handlers accept `group` but not the
adjacent `SortPriority` field on `UMaterialExpressionParameter`,
which controls UI ordering inside a group. Minor but lives in the
same domain.

## Proposed RPCs

```
material.authoring.clear_parameter_override
    assetPath, parameterName, parameterType ("scalar"|"vector"|"texture"|"staticSwitch")
    → { cleared: bool, inheritedValue: <typed> }

material.authoring.set_static_switch_parameter_value
    assetPath, parameterName, value (bool), save? (default true)

material.authoring.set_material_instance_parent
    assetPath, parentMaterial, preserveOverrides? (default true)

material.authoring.get_material_instance_info
    assetPath
    → { parent, overrides: {…}, inherited: {…}, parameters: [{name, type, group, sortPriority}] }

material.authoring.set_material_instance_parameters
    assetPath, scalar?: {Name: Value}, vector?: {…}, texture?: {…}, staticSwitch?: {…}, save?
    → { applied: [...], failed: [...] }
```

And add `sortPriority` (optional, default 32) to the existing
`add_scalar_parameter`, `add_vector_parameter`, `add_static_switch_parameter`,
`add_texture_sample` (when used as parameter) handlers.

## Why it matters

Material instances are how shipped games surface tweakable values — the
parent material is fixed, the instances are where artists work. Today,
an agent driving instance-based content authoring can write parameters
but can't:

- Verify what got written (no read-back)
- Undo a single override (no clear)
- Drive static-switch-gated visuals at all
- Re-parent instances when the parent hierarchy is restructured

All four are routine operations in any non-trivial material setup.

## History
- `#1-instance-api-incomplete` `OPEN` reporter — Material API audit found the material-instance surface covers only the override-write path (`create_material_instance`, `set_scalar/vector/texture_parameter_value`). Missing: `ClearParameterValueEditorOnly` (per-override reset), `SetStaticSwitchParameterValueEditorOnly` (static switch overrides), `SetParentEditorOnly` (parent reassignment), a read-side `get_material_instance_info` that returns parent + overrides + inherited values, and a batch parameter setter to amortize per-call shader recompile cost. `get_material_info` rejects `UMaterialInstanceConstant` with `UNSUPPORTED_ASSET_CLASS` so there is currently no way to read instance state at all. Also adds `SortPriority` to the parameter expression authoring handlers, which currently only expose `Group`.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Re-audited `MaterialAuthoringHandler.cpp`: confirmed only 4 instance RPCs exist (`create_material_instance` L1606, `set_scalar/vector/texture_parameter_value` L1663/L1702/L1751). Zero references to `ClearParameterValueEditorOnly`, `SetStaticSwitchParameterValueEditorOnly`, `SetParentEditorOnly`, or `StaticParameters` in the entire file. `get_material_info` L2131 does explicitly reject `UMaterialInstanceConstant` with `UNSUPPORTED_ASSET_CLASS` at L2147 (ticket cites L2143–2152). `SortPriority` absent from `add_scalar_parameter` L689, `add_vector_parameter` L721, `add_static_switch_parameter` L766. No duplicate board entries — other material tickets cover unrelated graph-edit gaps. Severity High justified (read-back gap blocks any verification workflow; static-switch absence blocks a routine authoring pattern). Keep bundled rather than splitting: all 6 gaps share one handler file + one UE class, dev would implement together, numbered sub-items already give per-capability granularity.
- `#3-mi-api-completion` `IN-REVIEW` developer — Added five new handlers in MaterialAuthoringHandler.cpp: clear_parameter_override (per-parameter removal from ScalarParameterValues/VectorParameterValues/TextureParameterValues + StaticParameters.StaticSwitchParameters; returns inherited default), set_static_switch_parameter_value (wraps SetStaticSwitchParameterValueEditorOnly inside FMaterialInstanceParameterUpdateContext for one permutation rebuild), set_material_instance_parent (SetParentEditorOnly + optional clear-of-overrides), get_material_instance_info (parent + overrides+inherited per type + parameter metadata, mirrors get_material_info shape with UNSUPPORTED_ASSET_CLASS guard), set_material_instance_parameters (batch via update context). Added optional sortPriority param to add_scalar_parameter / add_vector_parameter / add_static_switch_parameter (UPROPERTY SortPriority defaults to 32). Reshape note: ticket's cited single-parameter ClearParameterValueEditorOnly does not exist in UE 5.6; per-override clear implemented by RemoveAll on the override TArrays + StaticParameters set + UpdateStaticPermutation. Regression test TestMaterialInstanceOverrides.cpp exercises set/get/clear round-trip with overrides.scalar absence after clear.
- `#4-returned-missing-sortpriority` `OPEN` tester — Returned: `get_material_instance_info` read back parent, `overrides.scalar.Roughness=0.8999999761581421`, and `inherited.scalar.Roughness=0.5`, but the returned `parameters[0]` omitted the claimed `sortPriority` metadata after creating the parent parameter with `sortPriority: 7`. Test: `material.authoring.create_material` `/Game/__EA_Verify/M_McpVerify_MIOverrides_20260515`, `material.authoring.add_scalar_parameter` `Roughness` with `sortPriority: 7`, `material.authoring.create_material_instance`, `material.authoring.set_material_instance_parameters`, `material.authoring.get_material_instance_info`; `material.authoring.clear_parameter_override` returned `cleared=true` and `inheritedValue=0.5`.
- `#5-parameter-sortpriority-readback` `IN-REVIEW` developer — Fixed get_material_instance_info parameter metadata to emit per-parameter sortPriority from UMaterialInterface::GetParameterSortPriority independently of group metadata, so ungrouped parent parameters created with sortPriority (for example 7) round-trip in parameters[]. Added regression coverage in TestMaterialInstanceOverrides.cpp for Roughness sortPriority readback.
- `#6-verify-sortpriority-readback` `DONE` tester — Verified: created `/Game/__EA_Verify/M_McpVerify_MIOverrides_F2` with `add_scalar_parameter Roughness, defaultValue=0.5, sortPriority=7`, created instance `MI_McpVerify_F2`, then `get_material_instance_info` returned `parameters: [{name:"Roughness", type:"scalar", sortPriority:7}]` plus `inherited.scalar.Roughness=0.5`. Temp assets deleted via `asset.delete`.
- `#7-regression-static-switch-no-persist` `OPEN` reporter — Regression: gap #2 (static-switch parameter override) was implemented in `#3` and the feature marked DONE, but the static-switch override does NOT round-trip — it is a silent success-with-no-effect. The DONE verification (`#4`/`#6`) only exercised the scalar override and sortPriority paths, never the static switch, and the regression test cited in `#3` ("exercises set/get/clear round-trip with overrides.**scalar** absence after clear") does not cover static switch either, so the break was never caught. Repro (live, on parent `/Game/Materials/M_WeatheredMetal` with static switch `EnableRust` and instance `/Game/Materials/MI_RustyMetal`): `material.authoring.set_static_switch_parameter_value {assetPath:"/Game/Materials/MI_RustyMetal", parameterName:"EnableRust", value:true, save:true}` returns success `{parameterName:"EnableRust", value:true}` with no error — but `material.authoring.get_material_instance_info` then reports `overrides.staticSwitch: {}` (empty) and `flags.bHasStaticPermutationResource: false`, i.e. the override never landed in `StaticParameters.StaticSwitchParameters` (the getter reads `GetStaticParameters().StaticSwitchParameters` filtered on `bOverride`, the canonical store the setter writes via `SetStaticSwitchParameterValueEditorOnly` inside `FMaterialInstanceParameterUpdateContext`). The analogous **scalar** setter on the same instance persists fine (`set_scalar_parameter_value RustIntensity=0.8` → `overrides.scalar.RustIntensity=0.8`). Two independent set+save+re-read cycles confirmed empty `overrides.staticSwitch` each time. Net: the per-instance static-switch toggle (the headline use case of gap #2 — "binary feature toggles on per-instance basis") is non-functional despite the success return. Likely cause to investigate: `SetStaticSwitchParameterValueEditorOnly` write not being flushed/committed by the `FMaterialInstanceParameterUpdateContext` dtor before save (or `bOverride` not being set), since `bHasStaticPermutationResource` stays false. A dedicated static-switch set→get round-trip regression test is needed.
- `#8-fix-static-switch-write-path` `IN-REVIEW` developer — Fixed the static-switch write path root-caused in #7. Both setters opened `FMaterialInstanceParameterUpdateContext UpdateCtx(Instance, ...)` then called `Instance->SetStaticSwitchParameterValueEditorOnly(...)` on the *instance*; the context dtor commits its own pre-write snapshot via `UpdateStaticPermutation` (UE 5.7 `MaterialInstance.cpp` ~L3977 snapshot ctor / ~L3987 commit dtor), silently discarding the direct write — so `overrides.staticSwitch` stayed `{}` and `bHasStaticPermutationResource` stayed false. #7's diagnosis was correct: the write went to the instance, not the set the dtor commits. Fix routes the override into the context's own static set: added anon-namespace helper `ApplyStaticSwitchOverrideToSet(FStaticParameterSet&, FName, bool)` (set-or-add with `bOverride=true`, mirroring `UMaterialInstance::SetStaticSwitchParameterValueEditorOnly`) and call it on `UpdateCtx.GetStaticParameters()` at both sites: single setter `material.authoring.set_static_switch_parameter_value` and the batch `material.authoring.set_material_instance_parameters` staticSwitch branch — the same defect, which #7 did not flag. Scalar/vector/texture paths untouched (not static parameters). File: `Source/EditorAutomationRpcGateway/Private/Handlers/Material/MaterialAuthoringHandler.cpp`. Test: added `FMaterialInstanceStaticSwitchRoundTripTest` (`EditorAutomationRpcGateway.material.instance.StaticSwitchOverrideRoundTrip`) in `Source/EditorAutomationRpcGateway/Private/Tests/Material/TestMaterialInstanceOverrides.cpp` — builds a parent material with an `EnableRust` static switch + child instance, drives the real `set_static_switch_parameter_value` then batch `set_material_instance_parameters` handlers, and asserts `overrides.staticSwitch.EnableRust` is present with the expected value via `get_material_instance_info` (fails with empty `staticSwitch` if the fix is reverted). Did not compile or run tests (later phase).
- `#9-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
