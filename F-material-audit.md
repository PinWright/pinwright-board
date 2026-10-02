---
id: F-material-audit
title: "No material graph validation: nothing reports islands, null textures or functions, unused or duplicate parameters, or blend-mode/output mismatches"
status: IN-REVIEW
severity: Medium
category: feature
tags: [material, audit, validation, gap-analysis-2026-09-30]
encounters: 1
---

# No material graph validation (material.audit)

PinWright reports shader compile errors per permutation (`material.compile-state`, `ProbeAndWait` in `MaterialShaderState.h:543`), but nothing validates the graph. `asset.validate` only checks that an asset exists and loads. pinwright.com/compare rates "Shader stats / graph validation" partial. Monolith is the only yes: `validate_material` (`MonolithMaterialActions.cpp:1852-2135`) walks back from every material property, CustomOutput nodes and MaterialAttributeLayers, then checks islands, a null `TextureBase->Texture`, a null `MaterialFunction`, unused parameters, duplicate parameter names, more than 200 expressions, and blend mode vs outputs: masked without OpacityMask, translucent without Opacity, opaque with unused Opacity/Refraction, post-process without Emissive, subsurface on the wrong model. Its `fix_issues` only deletes islands. ue-mcp `ValidateMaterial` (`MaterialHandlers_WholeMaterial.cpp:87`) checks orphans, broken refs and UV-width mismatches (`:161`).

**Fix:** a §18 audit, `material.audit` (`Audit/AuditFramework.h`: `FVerdict::DerivePass`, `failOn`, a check table, buckets that sum).
- Params: `assets[]` or `folder`, `checks`, `failOn`, `includeShaderCompile` (folds in `ProbeAndWait` errors).
- Checks: `island`, `null-texture`, `null-function`, `unused-param`, `duplicate-param`, `blend-output-mismatch`, `uv-width`, `expression-budget`.
- Findings carry `nodeId` so the caller can batch-remove through existing verbs. No `fix_issues`: an audit writes nothing.
- This is the validation half of the compare row; `B-material-stats-instruction-count-hardcoded` is the stats half.

**Acceptance:**
- Fixtures with a known island, a null texture, and a Masked material with no OpacityMask each return `pass: false` with the right check id and `nodeId`.
- A clean material returns `pass: true`.
- Landscape materials, CustomOutput materials (grass, RVT) and layer-stack materials produce no false island findings.
- A typo in `checks` is an error.

Effort M. Risk: false positives from a missed reachability root.

## History
- `#1-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "Shader stats / graph validation": PinWright partial, Monolith yes). No graph-level validation exists. Evidence and design above.
- `#2-material-audit-verb` `IN-REVIEW` developer — Added `material.audit` (`Source/PinWright/Private/Handlers/Material/MaterialAuditHandler.cpp`), a section-18 audit on `Audit/AuditFramework.h`: params `assets` | `folder` (+`recursive`), `checks`, `failOn`, `includeShaderCompile`, `limit` (default 100; overflow sets `truncated`, which fails pass). Check ids are snake_case to match every other shipped audit (`island`, `null_texture`, `null_function`, `unused_param`, `duplicate_param`, `blend_output_mismatch`, `uv_width`, `expression_budget`, `shader_compile`), not the ticket's kebab spelling. Reachability roots: every `GetExpressionInputForProperty(MP_*)` input, every `UMaterialExpressionCustomOutput` (`GetAllCustomOutputExpressions`), named-reroute usage -> declaration; composite/pin-base/comment scaffolding is never reported. Severities: islands, unused params, ignored pins (`IsPropertyActiveInEditor`), too-wide UVs, budget are WARNINGS, so with the default `failOn:error` an island-only material passes; acceptance's island case is asserted under `failOn:any`. null_texture / null_function / too-narrow UVs are errors when reachable and warnings on an island; duplicate params are errors on a type clash, warnings on differing defaults, and identical copies (a shared parameter) are not flagged. Masked-without-OpacityMask and PostProcess-without-Emissive are errors; the raw `BlendMode` is read because `GetBlendMode()` reports mask-less Masked as Opaque. `blend_output_mismatch` is UNRUNNABLE on a material-attributes material (layer stacks, most landscape masters), so such a material fails `pass` unless the check is omitted — honest per section 18, but callers auditing attribute workflows will hit it; a MakeMaterialAttributes-aware evaluation would narrow it. No fix mode. New codes in `Handlers/ErrorCodes.h` (`MATERIAL_AUDIT_*`, 12). Version guard: `GetOutputValueType` (5.6+) vs `GetOutputType` (row added to `docs/engine-version-support.md`). Docs: `docs/wiki-src/material.md` (`### material.audit` + How-to-use mention), CHANGELOG. Tests `Tests/Material/TestMaterialAudit.cpp`, filter `PinWright.material.audit.`: `IslandIsFlaggedOnItsNode`, `NullTextureIsAnErrorOnItsNode`, `NullFunctionIsAnErrorOnItsNode`, `MaskedWithoutOpacityMaskFails`, `OpaqueWithOpacityPinIsAWarning`, `UnusedAndConflictingParametersAreFlagged`, `WideCoordinatesIntoA2DTextureAreFlagged`, `ExpressionBudgetIsFlagged`, `CleanMaterialPasses` (assets + folder mode, buckets sum), `CustomOutputAndRerouteRootsAreNotIslands` (landscape grass output, named reroute, landscape layer blend), `LayerStackMaterialHasNoFalseIslands`, `BadArgumentsAreErrorsAndMissingAssetsUnrunnable`. Compile-checked with UBT -SingleFile on 5.8 only; not built or run.
