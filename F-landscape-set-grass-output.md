---
id: F-landscape-set-grass-output
title: "Wiring a ULandscapeGrassType into a material's LandscapeGrassOutput has a generic route (`material.graph.add_expression` `properties` / `property.set` `GrassTypes[N].GrassType`) that is unverified live and documented nowhere — `landscape.create_grass_type` tells the caller to make that link without saying how"
status: OPEN
severity: Low
category: feature
tags: [landscape, grass, vegetation, material-graph, grass-output, node-property, coverage-gap, stranded-output, create_grass_type]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
rice: [1, 2, 0.5, 1]
priority: 8
---

# Verify and document the route from a grass type to the landscape material's grass output

A `ULandscapeGrassType` renders only when a `UMaterialExpressionLandscapeGrassOutput` in the
landscape material lists it in `GrassTypes[n].GrassType`. `landscape.create_grass_type`'s summary ends
with *"Reference this asset from a landscape grass node in the landscape material to render grass"*
(`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:1614`) and nothing in the surface
says how.

The generic route exists in source but has not been exercised live for this node:

- `material.graph.add_expression` takes a `properties` map
  (`Source/PinWright/Private/Handlers/Material/MaterialGraphHandler.cpp:524`), resolved by name on the
  expression class and written through `ApplyJsonValueToProperty`
  (`Source/PinWright/Private/Material/MaterialExpressionFactory.cpp:310-323`, `:344-358`), so
  `expressionClass: "LandscapeGrassOutput", properties: {GrassTypes: [{Name, GrassType}]}` should
  create the node already configured.
- `property.set` resolves array-subscript paths (`docs/wiki-src/property.md:49`), so
  `GrassTypes[0].GrassType` on an existing node's object path should write it; the factory sets the
  expression's `Material` back-pointer (`MaterialExpressionFactory.cpp:333-339`) so the edit reaches
  the material.
- `material.graph.connect_nodes` (`MaterialGraphHandler.cpp:149`) then wires a weight source
  (e.g. a `LandscapeLayerSample`) into the entry's input pin.

Unknowns: whether the JSON-to-`FGrassInput` array conversion accepts that shape, what pin name
`connect_nodes` expects for entry N, whether `material.graph.get_node_details` (`:355`) shows the
configured `GrassTypes`, and whether the landscape's grass maps rebuild after the material edit
(the refresh behaviour documented in `docs/wiki-src/landscape.md` § *Making a grass-type edit reach
the screen*).

**Workaround:** `python.execute` against `UMaterialExpressionLandscapeGrassOutput` (used in the
originating session).

**Fix:** run the route above live on a landscape material; if it works, document it in
`landscape.md` (grass section) and point `create_grass_type`'s summary at it. Add a dedicated
`landscape.set_grass_output` verb only if a step fails live, and file the failing step as its own
defect.

**Acceptance:** on a test landscape, `add_expression` (or `add_expression` + `property.set`) plus
`connect_nodes` produce a grass output whose `GrassTypes[0].GrassType` reads back as the created
asset, and grass instances appear on the painted layer without `python.execute`; `landscape.md` and
the `create_grass_type` summary name the exact calls.

## History
- `#1-grass-output-unreachable` `OPEN` reporter — Found while building a vegetation zone against a
  live editor. Re-derived by grep across `Plugins/PinWright/Source/PinWright/` (all `.cpp`/`.h`):
  zero hits for `LandscapeGrassOutput`, `GrassTypes`, `FGrassInput`, so no verb reads or writes the
  grass output node. The `material.graph` namespace has ten verbs
  (`MaterialDiscoveryHandler.cpp:200`, `:317`; `MaterialGraphHandler.cpp:35`, `:94`, `:134`, `:228`,
  `:327`, `:410`, `:480`, `:532`) and none of them sets a node property, so the node can be added
  and not configured. `FGrassInput.Input` proved unreadable from Python — it took a `python.execute`
  array round-trip. The stranded verb is `landscape.create_grass_type`
  (`LandscapeHandler.cpp:1502`), whose own summary instructs the caller to reference the asset from
  a landscape grass node — the step the surface cannot perform. Also noted: no verb merges
  `GrassVarieties` into an existing type.
- `#2-rephrased` `OPEN` developer — Re-checked at PinWright `7230b41d`. The "unreachable / no verb can set it" headline was wrong: `material.graph.add_expression` accepts a reflected `properties` map (`MaterialGraphHandler.cpp:524`, applied via `ApplyJsonValueToProperty` at `MaterialExpressionFactory.cpp:344-358`) and `property.set` resolves `GrassTypes[N].GrassType` subscripts (`docs/wiki-src/property.md:49`). Dropped the "zero grep hits", "no property setter", and "`FGrassInput.Input` unreadable from Python" framing; retitled to verify the generic route live and document it, with a dedicated verb only if it fails. `F-pcg-set-node-property` and `B-grass-varieties-edit-does-not-reach-renderer` are now IN-REVIEW. Severity Medium -> Low: the remaining gap is discoverability on a rare path (re-raise if the live check fails).
