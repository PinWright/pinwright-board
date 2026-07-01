---
id: E-get-material-info-no-param-defaults
title: "material.authoring.get_material_info omits parameter default values + group/sortPriority"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [material, material-authoring, readback, docs]
---

# material.authoring.get_material_info omits parameter default values + group/sortPriority

`material.authoring.get_material_info` lists each material parameter as
`{name, type, nodeId}` only — it never echoes the parameter's **default
value**, nor its `group` / `sortPriority`. See
`MaterialAuthoringHandler.cpp:2691-2704` (the per-parameter loop; the
ticket originally cited `2612-2625`, which drifted as the same handler
grew `shadingModel`/`mainInputs[]` above it): the per-parameter
`FJsonObject` sets `name` (from `TryGetMaterialParameterName`), `type`
(`Expr->GetClass()->GetName()`), and `nodeId`
(`MaterialExpressionGuid`), and nothing else.

This breaks the natural "author then verify" round-trip. An agent that has
just created `add_scalar_parameter("Roughness", 0.4)` /
`add_vector_parameter("BaseTint", (0.9,0.6,0.3))` and wants to confirm the
defaults landed has no read-back surface that echoes them — it can confirm
the parameter *exists* and its *type*, but not that the default it set is
the default now stored. The caller is forced to infer correctness
indirectly (clean compile + the fact that the create call returned ok)
rather than reading the value straight back.

The data is trivially available on the same expression objects the handler
is already iterating: `UMaterialExpressionScalarParameter::DefaultValue`
(float), `UMaterialExpressionVectorParameter::DefaultValue` (FLinearColor),
`UMaterialExpressionStaticBoolParameter::DefaultValue` (bool),
`UMaterialExpressionTextureSampleParameter::Texture` (UTexture*), plus the
shared `ParameterName` / `Group` / `SortPriority` metadata on
`UMaterialExpressionParameter`. The instance-side sibling
`get_material_instance_info` already returns `inherited` defaults and
`{name,type,group,sortPriority}` per parameter
(`material.authoring.md:54`), so the parent-material read surface is the
odd one out.

**What it should do:** add a `defaultValue` field plus `group` /
`sortPriority` (to match the instance read-back) to each entry in the
`parameters` array, dispatching on parameter expression subclass for the
correct typed value. Scalar → number, Vector → `{r,g,b,a}`, StaticBool →
bool, Texture → asset path. The `group`/`sortPriority` extraction is the
same logic `BuildExpressionDetailsJson` (`MGIR/MGIRExpressionUtils.h:246-
252`) and `get_material_instance_info` already emit, so reuse those field
semantics rather than inventing a new shape.

**Fix:** add a small `AddMaterialParameterDetails(ParamObj, Expr)` helper
beside `TryGetMaterialParameterName` in `MaterialAuthoringHandler.cpp`
that, for the parameter expression already in hand, sets `group` (when
not None) + `sortPriority` from `UMaterialExpressionParameter`, and a
typed `defaultValue` dispatched on subclass
(`UMaterialExpressionScalarParameter::DefaultValue` → number,
`UMaterialExpressionVectorParameter::DefaultValue` → `BuildLinearColorJson`,
`UMaterialExpressionStaticBoolParameter::DefaultValue` → bool,
`UMaterialExpressionTextureSampleParameter2D::Texture` → asset path); call
it from the per-parameter loop in `get_material_info`. This is a live-RPC
readback shape change (no asset-dump aspect, so no cache version bump).
Coordinate with IN-REVIEW `E-material-main-output-no-node-readback`, which
already landed `shadingModel`/`mainInputs[]` immediately above this loop
and edited the same `material.authoring.md` overlay — keep the docs note a
delta on top of that change.

**Docs angle:** until/unless the field is added, the `material.authoring`
wiki overlay (`docs/wiki-src/material.authoring.md`, the `get_material_info`
workflow note at line 15 and the Instance API contract at line 49-54)
should state explicitly that `get_material_info` returns parameter
name/type/nodeId only and does **not** echo defaults — so callers do not
expect a `defaultValue` field and design their verification around it. The
overlay already documents the richer `get_material_instance_info` shape;
the parent-material asymmetry deserves a one-line callout.

**Workaround:** confirm parameter creation succeeded via the create call's
own return + a clean `compile_material`, treating "compiles clean with the
expected parameter present" as a proxy for "default value is correct."
Heavier alternative: `material.decompile_mgir` and read the default out of
the IR.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean material.authoring.auto_layout task (focus material.authoring.auto_layout, outcome clean). Friction note: "get_material_info reports param names/types but does not echo default values, so defaults are confirmed via inputs+clean compile, not an echoed value field." Task built M_TintedPanel with 3 params (Roughness=0.4, TintStrength=0.75, BaseTint=(0.9,0.6,0.3)); story step 9 explicitly asked to "confirm the three parameters are present with their defaults," which the read-back could only partially satisfy (names/types yes, defaults no). Source-confirmed at `MaterialAuthoringHandler.cpp:2612-2625` — per-parameter JSON carries only name/type/nodeId; `DefaultValue` is on the expression subclasses being iterated and is not read. Sibling `get_material_instance_info` already returns defaults, making the parent-material read surface inconsistent. Low severity (workaround via create-return + clean compile exists); E-/docs because it is a read-back ergonomic + wiki-expectation gap, not a wrong result.
- `#2-additional-group-field` `OPEN` reporter — Additional evidence: independent realism task (painted-metal kit) hit the same gap, but on the **`group`** field specifically rather than defaults. Built M_PaintedMetal with BaseTint(vector, group "Appearance"), Roughness(scalar, "Appearance", 0.4), EmissiveBoost(scalar, "FX", 0.0). Replay-confirmed: `get_material_info(/Game/Materials/M_PaintedMetal)` returned `parameters:[{"name":"BaseTint","type":"MaterialExpressionVectorParameter","nodeId":"46AD..."},{"name":"Roughness","type":"MaterialExpressionScalarParameter","nodeId":"D361..."},{"name":"EmissiveBoost","type":"MaterialExpressionScalarParameter","nodeId":"9FB6..."}]` — no `group`, despite each param having been created with an explicit non-default group. To confirm the Appearance/FX grouping the agent had to fall back to **3 extra** `get_material_node_details` calls (each of which DOES carry `group`/`sortPriority`/typed default — e.g. BaseTint node returned `"group":"Appearance","sortPriority":32`). The child instance read-back is again the consistent one: `get_material_instance_info(MI_PaintedMetal_Red)` returned `parameters:[{"name":"Roughness","type":"scalar","group":"Appearance","sortPriority":32},{"name":"EmissiveBoost","type":"scalar","group":"FX","sortPriority":32},{"name":"BaseTint","type":"vector","group":"Appearance","sortPriority":32}]`. Reinforces #1's proposed fix: adding `group`/`sortPriority` (not just `defaultValue`) to each `get_material_info` parameter entry would close the asymmetry and eliminate the N-extra-call group check.
- `#3-reword-and-fix` `IN-REVIEW` developer — Reworded (title broadened to include group/sortPriority; refreshed the stale `2612-2625` citation to `2691-2704`; added a concrete **Fix:** section). Implemented in `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp`: added a `AddMaterialParameterDetails(ParamObj, Expr)` helper beside `TryGetMaterialParameterName` that sets `group` (only when not None, matching the instance read-back) + `sortPriority` from `UMaterialExpressionParameter`, and a typed `defaultValue` dispatched on subclass — scalar→number (`ScalarParameter::DefaultValue`), vector→`{r,g,b,a}` (`VectorParameter::DefaultValue` via `JsonBuilders::BuildLinearColorJson`), static bool/switch→bool (`StaticBoolParameter::DefaultValue`), texture→asset path (`TextureSampleParameter2D::Texture`); wired it into the `get_material_info` per-parameter loop (now `~2691-2710`), added the `MaterialExpressionStaticBoolParameter.h` include, and updated the handler summary. Live-RPC readback only — no asset-dump aspect, so no cache version bump. Rebased on IN-REVIEW `E-material-main-output-no-node-readback`'s `shadingModel`/`mainInputs[]` (already landed above this loop); docs note added as a delta to the step-5 workflow line in `Docs/wiki-src/material.authoring.md` describing the new `{name,type,nodeId,sortPriority,defaultValue}` (+`group`) shape. Regression test: `Source/PinWright/Private/Tests/Material/TestMaterialInfoParameterDefaults.cpp` (`PinWright.material.authoring.get_material_info.ParameterDefaults`) builds a transient material with a scalar (0.4, group "Appearance", sort 10), a vector ((0.9,0.6,0.3,1), "Appearance", 20), and a static switch (true, "FX", 30) and asserts `get_material_info` echoes each typed `defaultValue` + `group` + `sortPriority`; reverting the `AddMaterialParameterDetails` call fails it.
