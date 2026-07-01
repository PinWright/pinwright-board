---
id: F-search-api-material-expressions
title: "No catalog / keyword search for material expression node types"
status: DONE
severity: Medium
category: feature
tags: [search, material, material-graph, material-authoring, node-discovery]
---

# No catalog / keyword search for material expression node types

`material.graph.add_node` / `material.graph.add_expression` accept any
`UMaterialExpression*` class name (short form `Add` or long form
`MaterialExpressionAdd`), but there is **no RPC that enumerates the
available expression classes**. The asymmetry vs Blueprint is the issue:

- **Blueprint side:** `blueprint.graph.list_node_types` returns the K2Node
  catalog for a given asset; `blueprint.build_api_index` + `search_api`
  returns the function catalog. Two discovery surfaces, both typed.
- **Material side:** the wiki page for `material.authoring` lists a finite
  set of **convenience** adders (`add_math_node`, `add_panner`, `add_fresnel`,
  `add_voronoi`, …) and that markdown is the only catalog. For anything not
  in that list — and there are roughly 200+ `UMaterialExpression*` subclasses
  in stock UE 5.6 — the caller must already know the class name to feed
  `material.graph.add_node`. There is no `list_expression_types` and no
  ranked keyword search.

**Use cases blocked:**

1. "I need a node that does X" — e.g. "blue noise", "screen-space UV",
   "scene texture lookup", "Bumpoffset". Today: read engine source for
   `MaterialExpression*.h`, or trial-and-error class names against
   `add_node`.
2. Round-trip authoring from a high-level spec — an agent that knows it
   wants a `Fresnel` + `LightVector` + `CameraVector` pipeline cannot
   verify those expression classes exist without guessing or invoking
   `python.execute`.
3. Surface-area parity for cross-domain authoring — Blueprint, Niagara, and
   Material all use the same "imperative graph editor" mental model, and
   `blueprint.graph.list_node_types` exists for one of the three. Materials
   should match.

**Current workarounds:**

- `python.execute` enumerating `unreal.MaterialExpression` subclasses via
  reflection.
- Read `Engine/Source/Runtime/Engine/Classes/Materials/MaterialExpression*.h`
  on disk (~200 files).
- Grep the `material.authoring` wiki page for the named convenience adders
  and hope the desired node is among them.

All bypass the typed-RPC surface.

**Proposal:** Add `material.graph.list_expression_types` (catalog) and
`material.graph.search_expression_types` (keyword-ranked), mirroring the
existing Blueprint discovery pair:

```
material.graph.list_expression_types(
    category?: string,           // restrict to UE expression categories ("Math", "Texture", "Coordinates", "Vectors", "Parameters", "Utility", "Custom")
    domainFilter?: string,       // only include expressions valid for this MaterialDomain ("Surface", "PostProcess", "UI", ...)
    includeAbstract?: bool       // default false
) -> {
    expressions: [{
        className: "MaterialExpressionMultiply",
        shortName: "Multiply",
        category: "Math",
        description: "Multiplies two inputs component-wise",  // from class metadata
        inputPins: ["A", "B"],
        outputPins: [""]         // or named outputs where applicable
    }],
    count: number
}

material.graph.search_expression_types(
    query: string,
    category?: string,
    limit?: number               // default 20
) -> {
    results: [{ ...same shape as above..., score: number }],
    totalMatches: number
}
```

Implementation surface: walk `TObjectIterator<UClass>` once filtered to
`UMaterialExpression` descendants, cache name + `GetCaption()` +
`GetCategory()` + pin layout via a `UMaterialExpression*` CDO (each
expression's default object has its `ExpressionInput` UPROPERTYs and
returns pin metadata via `GetExpressionName` / `GetCaption`). The wiki page
for `material.authoring` already contains the manually curated subset; the
typed RPC would surface the full set.

**Bonus / optional:**

- Include "is parameter type" flag so callers can quickly find parameter
  expressions (`Scalar`, `Vector`, `Texture`, `StaticBool`).
- Include category-aligned defaults for common authoring (e.g. `Math/Add`
  defaults to pins `A` / `B`) so the result is directly consumable by
  `material.graph.add_node` callers without a separate `get_node_details`
  round-trip.

**Cross-ref:** If the broader
[`F-search-api-native-uclasses`](F-search-api-native-uclasses.md) ticket
lands first, this could become a thin wrapper that delegates to
`system.inspect.search_classes(parentClass="MaterialExpression")`. Both
surfaces are useful: the material-specific one returns pin metadata that
generic class search would not.

## History
- `#1-no-material-expression-catalog` `OPEN` reporter — Surveying discovery RPCs against `material.graph.add_node` / `material.graph.add_expression` revealed no peer for `blueprint.graph.list_node_types` / `blueprint.search_api` on the material side. The `material.authoring` wiki page is the only catalog, and it covers only the curated convenience adders, not the full ~200-class `UMaterialExpression*` taxonomy. Today the caller must already know the class name (e.g. `MaterialExpressionBumpOffset`) to feed `add_node`, or fall back on `python.execute`. Proposes `material.graph.list_expression_types` + `material.graph.search_expression_types` mirroring the Blueprint discovery pair, returning class name, category, description, and pin layout so results are directly consumable by `add_node`.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified by source-reading `Handlers/Material/MaterialGraphHandler.cpp` and `MaterialAuthoringHandler.cpp`: zero `list_*`/`search_*` discovery RPCs exist on the material surface (only `get_node_details` per-node, and `material.authoring.add_*` typed convenience adders). The asymmetry vs Blueprint (`blueprint.graph.list_node_types`, `blueprint.build_api_index` / `blueprint.search_api`) is real. Ticket fits the existing `F-search-api-*` family pattern (niagara-graph-nodes, metasound-nodes, anim-graph-nodes, niagara-modules, console-commands, native-uclasses) — scope, severity (Medium), and proposed RPC shape are consistent with siblings. Cross-ref to `F-search-api-native-uclasses` is appropriate; the material-specific pin-metadata payload is the differentiator that justifies a dedicated RPC rather than collapsing into the generic class search. No modifications required.
- `#3-discovery-rpcs-added` `IN-REVIEW` developer — Added Handlers/Material/MaterialDiscoveryHandler.cpp with material.graph.list_expression_types (filters: category/domainFilter/includeAbstract/parameterOnly) and material.graph.search_expression_types (ranked scoring with category filter and limit). Both walk TObjectIterator<UClass> filtered to UMaterialExpression. Regression test TestMaterialDiscoveryHandlers.cpp asserts count >= 50, Multiply has A/B inputs and Math category, Fresnel found, and search ranks "fresnel"/"multiply" top.
- `#4-verify-discovery-rpcs` `DONE` tester — Verified: material.graph.list_expression_types with category=Math/includeAbstract=false returned count=52 and MaterialExpressionMultiply with category=Math and inputPins A/B; material.graph.search_expression_types query=fresnel limit=5 returned MaterialExpressionFresnel as the only result with score=100.
