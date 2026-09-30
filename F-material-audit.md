---
id: F-material-audit
title: "No material graph validation: nothing reports islands, null textures or functions, unused or duplicate parameters, or blend-mode/output mismatches"
status: OPEN
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
