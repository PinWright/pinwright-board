---
id: E-material-expression-value-property-undocumented
title: "Literal-expression value-property keys (Constant.R, Constant3Vector.Constant) are undocumented; the add_expression properties examples point at DefaultValue, which literals reject with PROPERTY_NOT_FOUND"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-graph, add-expression, properties, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
rice: [1, 1, 1, 1]
priority: 8
---

# The value-property keys of literal material expressions (Constant.R, Constant3Vector.Constant) are undocumented

`material.graph.add_expression` takes an optional `properties` object of reflected
`UMaterialExpression` property names
(`Source/PinWright/Private/Handlers/Material/MaterialGraphHandler.cpp:524`).
`docs/wiki-src/material.graph.md:15` describes it with the example keys `ParameterName`,
`DefaultValue`, `Texture` and "vector/color structs". None of those is the value field of the
most common literal nodes:

- `MaterialExpressionConstant` holds its scalar in `R`, and `Constant2Vector` in `R` / `G`
  (`float`).
- `MaterialExpressionConstant3Vector` and `Constant4Vector` hold their color in `Constant`
  (`FLinearColor`).

A caller who follows the examples and passes `DefaultValue` on a `Constant3Vector` gets
`PROPERTY_NOT_FOUND: Property 'DefaultValue' was not found on MaterialExpressionConstant3Vector`
(`Source/PinWright/Private/Material/MaterialExpressionFactory.cpp:318`, `:428`). The error does
not list the class's settable properties, so the caller has to read the engine header to find
`Constant` / `R`. Repro (History `#1`): the M_WeatheredBronze build needed reads of
`MaterialExpressionConstant3Vector.h` and `MaterialExpressionConstant.h` to set two literals.

**Fix:** In `docs/wiki-src/material.graph.md:15`, add one sentence. Literal nodes take
`R` (`Constant`), `R`/`G` (`Constant2Vector`) and `Constant` as an `FLinearColor`
(`Constant3Vector`/`Constant4Vector`); `DefaultValue` belongs only to the `*Parameter`
expressions. Optionally, make `PROPERTY_NOT_FOUND` list the class's editable non-input
properties.

**Acceptance:** `material.graph.md` names `R` and `Constant` as the literal-node value keys.
`add_expression {expressionClass:"Constant3Vector", properties:{Constant:{R:0.55,G:0.38,B:0.18,A:1}}}`
works as documented.

**Workaround:** Read the engine header `MaterialExpression<Class>.h` for the UPROPERTY name.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced as PROCESS friction in a clean material.graph task (namespace material.graph, outcome clean) building /Game/Materials/M_WeatheredBronze: Constant3Vector(0.55,0.38,0.18)->BaseColor, Metallic param->Metallic, Multiply(Roughness param, Constant 0.8)->Roughness, plus a GradientTexture0 TextureSample. All 6 nodes wired and verified first-try (no retries, no is_error). Only friction: to set the literal values in the creating `add_expression` calls, the caller read engine headers `MaterialExpressionConstant3Vector.h` (FLinearColor `Constant`) and `MaterialExpressionConstant.h` (float `R`) because the `material.graph` wiki overlay's `properties`-keys note (`docs/wiki-src/material.graph.md` L7) names only `ParameterName`/`DefaultValue`/`Texture`/"vector/color structs" — none of which is the literal nodes' value field, and `DefaultValue` actively misleads (it's the *parameter* expressions' field; setting it on a literal does nothing). Source-confirmed: L7 advertises one-call property stamping but omits the two most common literal nodes' value keys. Distinct from E-material-pin-input-type-undiscoverable (connect-stage input pin type) and E-get-material-info-no-param-defaults (verify-stage default readback) — this is the author-stage value-UPROPERTY key. Propose docs-minimum (add Constant3Vector/Constant4Vector `Constant` and Constant/Constant2Vector `R`/`G` to the L7 paragraph, noting these are NOT `DefaultValue`); optional generalization via a `valueProperties[]` field on list/search_expression_types. Low — recoverable, no retries, but an engine-header detour for two of the most-used literal nodes.
- `#2-rephrased` `OPEN` developer — Rephrased. Dropped the claim that `DefaultValue` silently does nothing on a literal node. An unknown key is refused before insertion with `PROPERTY_NOT_FOUND` (`MaterialExpressionFactory.cpp:318`, `:428`; documented at `material.graph.md:15`), but the error lists no valid keys. Corrected the doc citation from L7 to `material.graph.md:15`. Dropped the `valueProperties[]` discovery field and replaced it with an optional candidate list in the error. Severity unchanged (Low, docs). RICE C 0.8->1 (verified in source and engine headers).
