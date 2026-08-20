---
id: B-shading-model-readback-domain-derived
title: "The shadingModel readback is sourced from UMaterial::GetShadingModels(), which is domain-derived — a PostProcess/UI/LightFunction material reads back Unlit and a DeferredDecal reads back DefaultLit no matter what was written"
status: OPEN
severity: Medium
category: bug
tags: [material, material-authoring, shading-model, readback, material-domain, decal, post-process, round-trip, engine-quirk, stale-readback]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# The write verb sets a member; the read verb reports a derived value

`Source/PinWright/Private/Handlers/Material/MainInputBindings.h:99` sources the `shadingModel`
readback field from `Material->GetShadingModels()`, and emits it at `:180`. Three consumers:
`MainInputBindings.h:180`, `AssetMaterialHandler.cpp:174`, `MaterialAuthoringHandler.cpp:3580`.

On UE 5.8.1 that function is domain-derived, not a member read —
`C:\UE_5.8\Engine\Source\Runtime\Engine\Private\Materials\Material.cpp:7462-7483`:

| `MaterialDomain` | `GetShadingModels()` returns |
|---|---|
| `MD_Surface`, `MD_Volume` | the stored `ShadingModels` bitfield |
| `MD_DeferredDecal`, `MD_RuntimeVirtualTexture` | hardcoded `MSM_DefaultLit` |
| `MD_PostProcess`, `MD_LightFunction`, `MD_UI` | hardcoded `MSM_Unlit` |
| anything else | `checkNoEntry()`, then `MSM_Unlit` |

Meanwhile `ApplyMaterialDomain` (`MaterialAuthoringHandler.cpp:555-572`) accepts `DeferredDecal`,
`LightFunction`, `Volume`, `PostProcess` and `UI`. So the plugin lets a caller build exactly the
materials on which the readback stops reflecting what was written:

- a **PostProcess / UI / LightFunction** material created with `shadingModel: "DefaultLit"` reads back
  `"Unlit"`;
- a **DeferredDecal** material always reads back `"DefaultLit"`, whatever was passed and whatever the
  `ShadingModel` UPROPERTY holds.

`get_material_info` is reporting the renderer's *effective* model while the write verb sets the
*stored* member, and for non-Surface/non-Volume domains those disagree with no indication.

`MaterialAuthoringHandler.cpp:2838` calls `Instance->GetShadingModels()` on a `UMaterialInstance` and
is unaffected — `MaterialInstance.cpp:5218-5221` is a flat `return ShadingModels;` with no domain
switch.

## The round-trip test already knows and dodges it

`TestMGIRMaterialPropertyRoundTrip.cpp:187` does assert
`TargetMaterial->GetShadingModels().HasOnlyShadingModel(MSM_ThinTranslucent)`, but `:106` sets
`MaterialDomain = MD_Surface` first — the branch that returns the stored field — so the assertion is
sound. The file states the trap in a comment at `:51-54` ("NOT what `GetShadingModels()` returns
(that one is derived per domain)") and reads the UPROPERTY by reflection via `ReadStoredShadingModel`
(`:55-60`) specifically to avoid it. No plugin test asserts a round-trip on a non-Surface domain, so
nothing would catch a regression here.

**Fix:** read the stored member by reflection (`FindPropertyByName(TEXT("ShadingModel"))`) for the
`shadingModel` readback field — `ReadStoredShadingModel` in the test file is the working
implementation — and, if the effective model is worth publishing, publish it as a separate,
differently-named field rather than conflating the two. Add a round-trip test on a non-Surface domain
so the distinction stays pinned.

## Related

- `E-material-main-output-no-node-readback` (IN-REVIEW) — the ticket whose fix **added** this
  `shadingModel` readback. This is a follow-on defect of that field, and is the natural place to
  attach the fix.
- The engine facts behind it are recorded as reference notes in `Docs/lessons.md` (domain-derived
  `GetShadingModels`, and the `SetShadingModel` precondition).

## History
- `#1-readback-is-effective-not-stored` `OPEN` reporter — `MainInputBindings.h:99` sources the `shadingModel` readback from `Material->GetShadingModels()` (emitted `:180`; consumers `MainInputBindings.h:180`, `AssetMaterialHandler.cpp:174`, `MaterialAuthoringHandler.cpp:3580`). On UE 5.8.1 that function is domain-derived, not a member read (`Material.cpp:7462-7483`): only `MD_Surface` and `MD_Volume` return the stored `ShadingModels`; `MD_DeferredDecal` and `MD_RuntimeVirtualTexture` return a hardcoded `MSM_DefaultLit`; `MD_PostProcess`, `MD_LightFunction` and `MD_UI` return a hardcoded `MSM_Unlit`. `ApplyMaterialDomain` (`MaterialAuthoringHandler.cpp:555-572`) accepts `DeferredDecal`, `LightFunction`, `Volume`, `PostProcess` and `UI`, so a caller can reach every affected domain from the wire: a PostProcess/UI/LightFunction material written with `shadingModel:"DefaultLit"` reads back `"Unlit"`, and a DeferredDecal always reads back `"DefaultLit"` regardless of what the UPROPERTY stores. The write verb sets the stored member while the read verb reports the renderer's effective model, and nothing marks the difference. `MaterialAuthoringHandler.cpp:2838` is unaffected — `UMaterialInstance::GetShadingModels()` (`MaterialInstance.cpp:5218-5221`) has no domain switch. The existing round-trip test is sound only because it pins `MD_Surface` first (`TestMGIRMaterialPropertyRoundTrip.cpp:106`, assertion `:187`); the file documents the trap at `:51-54` and reads the UPROPERTY by reflection via `ReadStoredShadingModel` (`:55-60`) to dodge it, and no plugin test asserts a round-trip on a non-Surface domain. Fix: read the stored member by reflection for the readback field (reuse `ReadStoredShadingModel`), publish the effective model separately if it is wanted, and add a non-Surface round-trip test. Follow-on defect of `E-material-main-output-no-node-readback` (IN-REVIEW), whose fix introduced the field.
