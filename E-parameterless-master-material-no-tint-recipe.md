---
id: E-parameterless-master-material-no-tint-recipe
title: "get_material_info answering `parameters: []` is a dead end: it is the correct answer, it means no material instance can adjust this master, and neither the response nor any page names the three routes out — while the one route that would be cheapest, promoting an existing expression to a parameter, is the one verb the namespace does not have"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-authoring, get_material_info, parameters, material-instance, recipe, docs, discoverability, shared-content, promote-to-parameter, premise-corrected]
encounters: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# The readback is right, and it leaves the caller at a wall with no signposts

`material.authoring.get_material_info` on
`/Game/ExampleContent/Landscapes/Materials/M_Rock` returns 28 nodes and `parameters: []`. That is a
true and useful answer: a master with no parameters cannot be adjusted through a material instance,
because there is nothing for an instance to override. It is also the whole of what the caller gets.
No field, no warning, and no page says what to do next — and one of the obvious next moves is
actively wrong for the commonest case.

## Measured

Look-dev polish over `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 3). Under the level's
final key (sun 13.3 lux, sky 6.30) `M_Rock`'s albedo reads as featureless white; a hillside of rocks
was reported twice by reviewers as *"white spheres with a missing material"*. The material is not
broken — its albedo is simply too high for that key, which is exactly the kind of thing a material
instance exists to fix.

`parameters: []` closed that route. The pass retargeted 5 instanced components and 33 actor
components onto a different material (`M_ForestRock`) instead — the right call, and one it reached by
elimination rather than by anything the tool said.

## Premise correction — "no verb offers an alternative path" is false

The finding as reported claimed there was no path short of duplicating and re-authoring a 28-node
graph. That does not survive an inventory of the namespace. Adding a tint to an existing main input
is **three calls**, all of which exist:

| step | verb | file:line |
|---|---|---|
| add a `VectorParameter` | `material.authoring.add_vector_parameter` | `Plugins/PinWright/Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:1153` (creates `UMaterialExpressionVectorParameter` at `:1197`) |
| add a `Multiply` | `material.authoring.add_math_node` | `:1240`, `Multiply` branch at `:1264-1265` |
| wire it into BaseColor | `material.authoring.connect_nodes` | `:1698`, `targetNodeId` empty or `'Main'` for a main material input (`:1703`, branch at `:1727`) |

The `material.graph.*` siblings do the same (`MaterialGraphHandler.cpp:480` generic
`add_expression`, `:134` `connect_nodes`, whose own summary documents the `'Main'` sentinel). So the
capability the finding asked for is shipped, and the ticket has to be about something narrower.

## What survives, and it is three separate things

**1. There is no promote-to-parameter verb, and the plugin's own text confirms the workaround.**
Nothing in `material.*` converts an existing expression into a parameter. The documented alternative
is to delete and rebuild, stated by the MGIR emitter itself:

```cpp
TEXT("with material.graph.remove_node + add_expression + connect_nodes. If it is a ")
TEXT("parameter expression, material.authoring.set_*_parameter_value edits it in place."),
```
`Plugins/PinWright/Source/PinWright/Private/MGIR/MGIRExpressionEmitter.cpp:184-185`

The asymmetry is the point: once an expression *is* a parameter it can be edited in place; getting it
to be one means removing and re-adding it, and re-wiring whatever it fed. For the common intent —
"the constant driving Roughness should have been a parameter" — that is the difference between one
call and a subgraph rebuild.

**2. `parameters: []` names no next step.** The response says what is absent and not what to do about
it. This is the same verb as `E-get-material-info-no-param-defaults` (IN-REVIEW, Low), which asks it
to say more about the parameters that *are* there; this asks for one line about the case where there
are none.

**3. The route a caller is most likely to take is the wrong one for shared content, and nothing
warns.** `M_Rock` lives under `/Game/ExampleContent/` and is used by other maps. Inserting a
`Multiply` + `VectorParameter` into it — the three calls above, which all succeed — changes every
consumer in the project. The correct move for a shared master is `asset.duplicate` then edit the
copy, or retarget the consumers, and neither is mentioned anywhere near a readback that has just
told the caller their only lever is the graph itself. **The three verbs make the wrong thing easy
and say nothing.**

## Ask

A short recipe, on `Docs/wiki-src/material.authoring.md` and referenced from the
`get_material_info` notes, covering the `parameters: []` case in the order a caller should consider
it:

1. **Retarget** the consumers to a material that already has the lever you need — cheapest, no asset
   edited, and what the measured case did.
2. **`asset.duplicate` then edit the copy** — when the tint is specific to this level. State the
   shared-content hazard here explicitly: editing a master under `/Game/ExampleContent/` (or any
   path other maps reference) changes them too, and the graph verbs will not stop you.
3. **Edit the master in place**, with the three-call recipe spelled out (`add_vector_parameter` +
   `add_math_node{Multiply}` + `connect_nodes{targetNodeId:"Main"}`) — correct only when you own the
   material.

Optionally, and separately if it is judged worth a ticket of its own: a
`material.authoring.promote_to_parameter {materialPath, nodeId, parameterName}` that replaces a
constant expression with the matching parameter type and re-wires its consumers, which is the
mechanical part of (3) that callers currently hand-roll.

## Related

- `E-get-material-info-no-param-defaults` (IN-REVIEW, Low) — same verb, same `parameters` array,
  complementary ask (say more about present parameters vs. say something about their absence). Worth
  landing together.
- `E-material-usage-flags-unreadable` (OPEN, Medium) — the other `get_material_info` omission found
  in the same session.
- `F-base-material-param-default-setter` (OPEN) / `B-material-param-setters-wrong-class-error`
  (OPEN, Medium) — the adjacent problem one step later: once a master *does* have parameters,
  editing their defaults is its own detour.
- `F-material-instance-overrides-incomplete` — the instance side of the same wall.

## Severity

**Low.** Impact class is the rubric's Low band, verbatim: *"pure friction. Docs, discoverability,
naming"*. Nothing is wrong. `parameters: []` is the correct answer, the three authoring verbs work,
`asset.duplicate` exists, and the measured case reached a good outcome — it simply reached it by
elimination, having first concluded from the empty array that there was no path at all. That
conclusion was wrong, which is the clearest evidence available that the discoverability gap is real:
the caller who hit it wrote down "no verb offers an alternative path" about a namespace that ships
three.

**Considered and rejected: Medium.** The argument would be that item 3 above — the easy wrong move on
shared content — is a hazard rather than friction, since the three calls succeed and silently change
other maps. It is rejected because no PinWright surface claims otherwise and the verbs are behaving
exactly as documented; a caller editing a shared asset is doing a normal thing whose consequences the
plugin does not currently comment on. If a future ticket asks the material writers to warn when they
mutate an asset under a shared content root, that would be the Medium one, and it would generalise
well past materials.

**Reach modifier declined in both directions.** `get_material_info` is a common verb, but
`parameters: []` on a master someone wants to tint is a specific situation, so no bump up; and it is
not a rare edge path either, since parameterless masters are normal in engine and marketplace
content, which is exactly where a project borrows materials from. Low stands unmodified.

## History
- `#1-empty-parameters-is-a-dead-end` `OPEN` reporter — Observed during the look-dev polish pass over
  `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 3):
  `material.authoring.get_material_info` on `/Game/ExampleContent/Landscapes/Materials/M_Rock`
  returned 28 nodes and `parameters: []`, so the rocks — reported twice by reviewers as "white
  spheres with a missing material" under the level's final key — could not be corrected with a
  material instance; the pass retargeted 5 instanced and 33 actor components to `M_ForestRock`
  instead. **PREMISE CORRECTED, and the correction is why this is filed Low rather than as a feature
  request:** the reported claim was that no verb offers an alternative path, and that is false —
  `material.authoring.add_vector_parameter` (`MaterialAuthoringHandler.cpp:1153`, expression created
  at `:1197`), `add_math_node` with the `Multiply` branch (`:1240`, `:1264-1265`) and
  `connect_nodes` into a main input via the `'Main'` sentinel (`:1698`, `:1703`, `:1727`; sibling
  `MaterialGraphHandler.cpp:134`, `:480`) express exactly the asked-for Multiply+VectorParameter
  insertion in three calls. WHAT SURVIVES, three things: (a) no promote-to-parameter verb exists,
  and the plugin's own MGIR emitter text (`Private/MGIR/MGIRExpressionEmitter.cpp:184-185`) documents the
  workaround as remove-then-re-add — so a parameter can be edited in place once it exists but
  becoming one costs a subgraph rebuild; (b) `parameters: []` names no next step, the mirror of
  `E-get-material-info-no-param-defaults` (IN-REVIEW, Low), which asks the same array to say more
  when it is non-empty; (c) the route the three verbs make easiest is wrong for the case that
  produced this — `M_Rock` is `/Game/ExampleContent/` shared with other maps, so inserting a tint
  node succeeds and changes every consumer, while the correct move is `asset.duplicate`-then-edit or
  retarget, and neither is mentioned near the readback. Asked for: a three-option recipe on
  `Docs/wiki-src/material.authoring.md` referenced from the `get_material_info` notes (retarget /
  duplicate-then-edit with the shared-content hazard stated / edit in place with the three-call
  spelling), plus optionally a separate `material.authoring.promote_to_parameter`. Rated **Low** —
  everything works and the measured case reached a good outcome; the evidence that the gap is real
  is precisely that the caller who hit it concluded "no verb offers an alternative path" about a
  namespace shipping three. **Medium considered and rejected**: the easy-wrong-move-on-shared-content
  reading is a hazard argument, but no surface claims otherwise and the verbs behave as documented —
  if that is to be a ticket it should be a general "warn when a writer mutates an asset under a
  shared content root", which generalises past materials. Reach declined both ways.
