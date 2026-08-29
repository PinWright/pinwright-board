---
id: F-landscape-set-grass-output
title: "No verb wires a ULandscapeGrassType into a material's MaterialExpressionLandscapeGrassOutput, so landscape.create_grass_type produces an asset nothing PinWright can build will ever render — GrassTypes[n].GrassType is unreachable and FGrassInput.Input is not even readable from Python"
status: OPEN
severity: Medium
category: feature
tags: [landscape, grass, vegetation, material-graph, grass-output, node-property, coverage-gap, stranded-output, create_grass_type]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The one link between a grass type and the landscape is the one link PinWright cannot make

A `ULandscapeGrassType` renders only when a `UMaterialExpressionLandscapeGrassOutput` node in the
landscape material points at it — `GrassTypes[n].GrassType`. Nothing in PinWright sets that
property, and nothing in PinWright reads that node.

Grepped across the whole plugin source (`Plugins/PinWright/Source/PinWright/`, all `.cpp` and `.h`):
**zero hits** for `LandscapeGrassOutput`, `GrassTypes`, and `FGrassInput`. The node type is not
referenced anywhere in the surface.

The generic escape hatches do not reach it either. The `material.graph` namespace is
`list_expression_types`, `search_expression_types`, `add_node`, `remove_node`, `connect_nodes`,
`break_connections`, `get_node_details`, `add_texture_sample`, `add_expression`, `create_nodes`
(`Handlers/Material/MaterialDiscoveryHandler.cpp:200`, `:317`;
`Handlers/Material/MaterialGraphHandler.cpp:35`, `:94`, `:134`, `:228`, `:327`, `:410`, `:480`,
`:532`) — it can add the node and wire pins, and it has **no property setter**. So a caller can
create the grass output node and cannot tell it which grass type to output.

There is likewise no verb to merge `GrassVarieties` into an existing `ULandscapeGrassType`.

## Why this is a feature ticket and not a wish

`landscape.create_grass_type` (`Handlers/Environment/LandscapeHandler.cpp:1502`) ships, is
documented, and its own summary ends by telling the caller to do a thing PinWright cannot do:

> "Create a ULandscapeGrassType asset configured to scatter the given static mesh as grass.
> **Reference this asset from a landscape grass node in the landscape material to render grass.**"

That second sentence is the whole gap. The verb can produce a perfectly valid grass type — correct
mesh, correct density, correct scales, correct cull distances after
`B-create-grass-type-addzeroed-never-renders` — and **nothing PinWright can build will ever render
it**, because the one link between the asset and the landscape material is unreachable through the
surface. The verb's output is stranded by construction.

## What the workaround actually costs

It is reachable through `python.execute`, and the zone agent did it in-session, so this is not "no
workaround". It is worth recording precisely what that path involves, because it is what keeps this
above Low:

- `FGrassInput.Input` is **not readable from Python** — reading it took an array round-trip through
  `python.execute`, not a property read. So the caller cannot inspect the node's current wiring
  before writing to it.
- Finding that out requires reading the engine's `UMaterialExpressionLandscapeGrassOutput`
  declaration. There is no discoverable path from any PinWright response to this node.

That is the Medium band's "a source dive" as the only route, on the finishing step of a workflow the
plugin otherwise supports end to end.

## Ask

Either of these closes it; the first is narrower and safer, the second is more useful.

1. **`landscape.set_grass_output`** — resolve the landscape material, find or add its
   `UMaterialExpressionLandscapeGrassOutput`, and set `GrassTypes[n]` (name + `GrassType` asset)
   for a named entry, creating the entry if absent. Report the resulting `GrassTypes` array read
   back off the node, not echoed, so an entry that will not render is visible in the response —
   the same measured-readback style `create_grass_type` already uses for `density` and
   `end_cull_distance` (`LandscapeHandler.cpp:1616-1617`).

   Whatever lands must also flush: a grass-output edit that does not reach the renderer is the
   defect in `B-grass-varieties-edit-does-not-reach-renderer` (OPEN, High), filed the same session,
   and this verb would be the newest way to hit it.

2. **`material.graph.set_node_property`** — a generic property setter on a material expression node,
   general enough to reach `GrassTypes[n].GrassType` among everything else. This is the bigger win:
   the `material.graph` namespace currently has ten verbs and no way to set a property on a node it
   has just added, so every node class with configuration lives behind the same wall.

   The board already carries the analogous ask for another graph namespace —
   `F-pcg-set-node-property` (OPEN, High) — whose reasoning transfers directly, and whose "partly
   covered by `property.set` on the sub-object's path" mitigation is **weaker here**, because a
   material expression's array-element struct field is what needs writing and `FGrassInput.Input`
   is already shown not to be readable through the reflection surface.

A merge verb for `GrassVarieties` on an existing type is a separate, smaller ask and is noted rather
than requested here.

## Same shape as

- `F-pcg-set-node-property` (OPEN, High) — the same gap, one namespace over: typed node setters
  exist, a generic one does not. Distinct: different namespace, different node hierarchy, and there
  the `property.set` mitigation genuinely works.
- `F-base-material-param-default-setter` (OPEN, Low) — a material property that is readable but has
  no setter. Same family of coverage gap, much smaller: that one strands a parameter default, this
  one strands a whole verb's output.
- `E-material-expression-value-property-undocumented` — the discoverability half of writing
  expression properties.
- `B-configure-layer-blend-wrong-nodes` — the landscape-material-authoring neighbour. Distinct: that
  is a typed verb producing the wrong nodes, this is the absence of any verb at all.

## Severity

**Medium**, by impact class. The rubric offers "High or Medium: hard blocker with no workaround (a
stub, a missing verb, or rejecting valid input), so a reasonable task is impossible", and "Medium:
soft blocker. Doable, but only via a documented workaround, a source dive, or many extra calls".

**High considered and declined.** The argument for High is real and is the ticket's headline: a
shipped verb's output is inert without this, which reads as "a reasonable task is impossible".
Declined because it is not impossible — a `python.execute` route exists and was exercised in this
session. The rubric's High band is keyed on "no workaround", and there is one. What the workaround
costs is a source dive, which is Medium's own wording, almost exactly.

**Reach modifier declined.** Landscape grass authoring is a rare path, which by the rubric would
bump this down to Low. Declined: Low is "pure friction — docs, discoverability, naming, cosmetic",
and this is not friction. Without it a verb PinWright ships cannot produce a visible result through
PinWright at all, which is a capability gap, not a papercut.

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
