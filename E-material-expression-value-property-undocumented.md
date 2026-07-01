---
id: E-material-expression-value-property-undocumented
title: "Literal-expression value-property names (Constant3Vector.Constant, Constant.R) are undocumented — add_expression properties keys force an engine-header read"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-graph, add-expression, properties, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# The value-property field name for literal expressions is undiscoverable

`material.graph.add_expression` accepts an optional `properties` object whose
keys are `UMaterialExpression` UPROPERTY names, and the `material.graph` wiki
overlay documents this well in the abstract — `docs/wiki-src/material.graph.md`
L7: *"common fields such as `ParameterName`, `DefaultValue`, `Texture`, or
vector/color structs can be set in the same call that creates the expression."*

But the two most basic literal expressions do **not** carry their value in any
of the named example fields, and the wiki names neither:

- `MaterialExpressionConstant3Vector` holds its color in `Constant`
  (type `FLinearColor`) — **not** `DefaultValue` (that field is on the
  *parameter* expressions) and **not** an obvious `Color`/`Value`.
- `MaterialExpressionConstant` holds its scalar in `R` (type `float`) — a
  single-letter field name that is impossible to guess from the documented
  examples.

So a caller wanting to stamp a literal RGB tint on a `Constant3Vector` and a
literal scalar on a `Constant` *in the same `add_expression` call* (the exact
ergonomic the L7 note advertises) has no documented field name to put in
`properties`. The example keys (`DefaultValue`, `Texture`, "vector/color
structs") actively mislead toward `DefaultValue`, which silently does nothing on
a non-parameter literal node.

## Why it's process friction (clean outcome, but an engine-header detour)

The task built `/Game/Materials/M_WeatheredBronze` cleanly (all 6 nodes wired
and verified first-try, no retries, no errors). The only friction was discovery:
to set the bronze tint on the `Constant3Vector` and `0.8` on the `Constant` in
the creating `add_expression` calls, the caller read the engine headers
`MaterialExpressionConstant3Vector.h` (→ `FLinearColor Constant`) and
`MaterialExpressionConstant.h` (→ `float R`) to learn the property keys.

Friction note verbatim: *"the wiki didn't state the Constant3Vector color
property name or the Constant scalar's field, so I read the UE engine headers
(MaterialExpressionConstant3Vector.h -> FLinearColor Constant;
MaterialExpressionConstant.h -> float R) to set them in one add_expression call
... normal investigation, no retries or errors."*

Net cost: an off-tool engine-source read for two of the most common literal
nodes, on an otherwise first-try graph build. Low severity (recoverable, no
retries), but it is exactly the engine-header detour the `properties`-keys
documentation is meant to eliminate, and the documented example keys point the
wrong way for literals.

## What it should do

Docs is the cheap, root-cause fix (the `properties`-keys note already exists —
it just omits the literal nodes):

- **Docs (minimum):** in `docs/wiki-src/material.graph.md`, extend the L7
  `add_expression` `properties` paragraph with the literal-node value-property
  names: *`Constant3Vector`/`Constant4Vector` carry their color in `Constant`
  (an `FLinearColor`); `Constant`/`Constant2Vector` carry scalar literals in
  `R` (and `G` for `Constant2Vector`) — these are NOT `DefaultValue`, which only
  applies to the `*Parameter` expressions.* Naming the two most-used literal
  nodes converts the engine-header read into a one-line lookup.
- **Discovery readback (optional, lower priority):** the existing
  `list_expression_types` / `search_expression_types` (see
  `F-search-api-material-expressions`, DONE) emit pin metadata but not the
  settable value-UPROPERTY names per class. A `valueProperties[]` field (the
  expression CDO's non-input UPROPERTYs that drive its output) would make the
  `properties`-key set discoverable for *any* expression, not just the curated
  literals — generalizing this docs fix.

Distinct from `E-material-pin-input-type-undiscoverable` (input *pin* type for
wiring/connecting — the connect stage) and from `E-get-material-info-no-param-defaults`
(reading parameter defaults *back* — the verify stage). This ticket is the
*author/create* stage: the value-holding UPROPERTY key you stamp at add time.
Wiki page to improve: `docs/wiki-src/material.graph.md`.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced as PROCESS friction in a clean material.graph task (namespace material.graph, outcome clean) building /Game/Materials/M_WeatheredBronze: Constant3Vector(0.55,0.38,0.18)->BaseColor, Metallic param->Metallic, Multiply(Roughness param, Constant 0.8)->Roughness, plus a GradientTexture0 TextureSample. All 6 nodes wired and verified first-try (no retries, no is_error). Only friction: to set the literal values in the creating `add_expression` calls, the caller read engine headers `MaterialExpressionConstant3Vector.h` (FLinearColor `Constant`) and `MaterialExpressionConstant.h` (float `R`) because the `material.graph` wiki overlay's `properties`-keys note (`docs/wiki-src/material.graph.md` L7) names only `ParameterName`/`DefaultValue`/`Texture`/"vector/color structs" — none of which is the literal nodes' value field, and `DefaultValue` actively misleads (it's the *parameter* expressions' field; setting it on a literal does nothing). Source-confirmed: L7 advertises one-call property stamping but omits the two most common literal nodes' value keys. Distinct from E-material-pin-input-type-undiscoverable (connect-stage input pin type) and E-get-material-info-no-param-defaults (verify-stage default readback) — this is the author-stage value-UPROPERTY key. Propose docs-minimum (add Constant3Vector/Constant4Vector `Constant` and Constant/Constant2Vector `R`/`G` to the L7 paragraph, noting these are NOT `DefaultValue`); optional generalization via a `valueProperties[]` field on list/search_expression_types. Low — recoverable, no retries, but an engine-header detour for two of the most-used literal nodes.
