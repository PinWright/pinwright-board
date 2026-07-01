---
id: E-level-bp-node-verbs-cant-author-bound-nodes
title: "level.structure.add_level_blueprint_node silently drops nodeName and reuses the nodeName response key for the auto-title; its dedicated level-BP authoring sub-family is an undocumented unbound-stub dead end vs blueprint.graph.*"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [level-structure, level-blueprint, blueprint-graph, nodeName, unbound-node, shim, docs]
---

# `level.structure.add_level_blueprint_node` drops `nodeName`, reuses the `nodeName` response key for the auto-title, and is an undocumented unbound-stub verb

The `level.structure` namespace ships a level-Blueprint authoring sub-family —
`open_level_blueprint`, `add_level_blueprint_node`, `connect_level_blueprint_nodes` —
that reads as the natural first-class path for "open the level BP and wire its
event graph." `add_level_blueprint_node` only ever emits an **unbound** stub
node (no event/function reference), so the bound-node authoring a caller
actually wants must be done with the generic `blueprint.graph.*` family on the
level-script BP object — but the verb advertises none of that, and on top of it
mishandles its own `nodeName` parameter.

Two concrete, low-risk ergonomic defects in `add_level_blueprint_node`
(`LevelStructureHandler.cpp:1706`) — these are the actionable core of this
ticket:

1. **`nodeName` is accepted but silently ignored, and the response key is
   reused.** The handler declares `RPC_PARAM_OPT("nodeName", "string", …)`
   (line 1711) and reads it into a local `NodeName` (line 1718) — then **never
   uses it**. The node is created with a bare
   `NewObject<UK2Node>(EventGraph, NodeClassObj)` (line 1786); the response then
   echoes back the *auto-generated node title* under the same key `nodeName`
   (line 1823, sourced from `NewNode->GetNodeTitle(...)` at line 1795). So a
   caller who passes `nodeName:"BeginPlay_Start"` gets a success body whose
   `nodeName` is `"Event None"` (or similar) — the param was dropped, and the
   echoed field misleadingly reuses the same key for a different value.

2. **It only emits an UNBOUND K2Node, and never says so.** The handler does
   `NewObject<UK2Node>(...)` + `AllocateDefaultPins()` with no event/function
   reference binding. For `UK2Node_Event` this yields a node with no
   `EventReference` (auto-title "Event None"), not a real `Begin Play`. The
   registration summary and the wiki never warn of this, so a caller discovers
   the dead end only by reading the handler C++.

The net process cost: a caller who follows the obvious same-namespace path
(`open_level_blueprint` → `add_level_blueprint_node`) cannot complete a trivial
"BeginPlay -> Print String" wire and must fall back to `blueprint.graph.*`
against the level-script object path.

## What it should do (re-scoped: honest-param + docs, not a new binding sub-family)

This ticket was **re-scoped down** from its original "give the verb real
event/function binding params." That half would re-implement
`blueprint.graph.create_node` (which already binds events via
`EventReference.SetFromField` using `eventName`, and which the verb's own
summary already points to) — gold-plating, not a defect. The dropped-param +
echo-shadow defects are independent of that and are the real fix:

- **Honor `nodeName` instead of throwing it away.** Apply the caller's
  `nodeName` to the created node (as its Comment label) so it is not silently
  dropped, and **echo it back under `nodeName`** in the response. Put the
  auto-generated node title under a separate `nodeTitle` key so the two values
  stop sharing one key.
- **Remove the `nodePosition` param-shape footgun.** Position already lands when
  passed as the documented object `{x,y}`, but the sibling
  `blueprint.graph.create_node` spelling (flat `x`/`y`) is silently zeroed.
  Accept flat `x`/`y` as an alias so an array/flat arg no longer pins the node
  at origin.
- **Document the unbound-stub limitation.** The registration summary and the
  wiki overlay must state that these verbs create unbound stub nodes only and
  steer callers to `blueprint.graph.create_node` on the level-script BP object
  path for real (bound) authoring.

## Docs angle (wiki)

`docs/wiki-src/level.structure.md` currently mentions level-BP editing only as
"Level-blueprint graph editing follows the same pattern as `call("blueprint.graph")`
— see that namespace's wiki" and never names `open_level_blueprint` /
`add_level_blueprint_node` / `connect_level_blueprint_nodes` at all. The overlay
should add a short note that the dedicated `level.structure.*` level-BP verbs do
**not** author bound event/call nodes, steering callers to
`blueprint.graph.create_node` on the level-script BP object path for real
authoring. (The companion handle-path trap — `open_level_blueprint` returning a
bare package `assetPath` that `blueprint.graph.*` rejects — is tracked
separately in `E-open-level-blueprint-unusable-assetpath`.)

## Evidence (process friction from a live task)

Task: open `/Game/Maps/ExampleProjectWelcome`, open its level BP, add a
BeginPlay event + Print String, wire them. Attempt-agent friction note:

> "the level-specific `level.structure.add_level_blueprint_node` can't bind an
> event/function ref or apply nodeName (handler ignores it, creates unbound
> K2Node), so it can't author a real BeginPlay->PrintString; I switched to
> blueprint.graph.create_node … Node internal names are auto-generated and there
> is no MCP rename, so the requested names were applied as node Comments. I had
> to read plugin C++ (LevelStructureHandler/BlueprintGraphCrudHandler) to learn
> nodeName is ignored and that Event needs eventName binding — a discoverability
> gap the wiki should cover."

Call-log shape (16 calls for a 2-node + 1-wire intent): after the level opened
and structure was read, the flow needed the generic `blueprint.graph.*` family
to do the real authoring (`create_node` x2 with `eventName` binding,
`connect_pins`, plus `set_node_property` Comment x2 to stand in for the
unavailable node rename), then 4 readback/`find_nodes` calls to confirm. The
dedicated `add_level_blueprint_node` / `connect_level_blueprint_nodes` verbs
were not usable for the bound-node author and were bypassed entirely.

## History
- `#3-honest-param-fix-and-docs` `IN-REVIEW` developer — Re-scoped down from the original "add real event/function binding params" (that half would duplicate `blueprint.graph.create_node`, which already binds events via `EventReference.SetFromField` using `eventName` and which the verb's own summary already points to — gold-plating, not a defect). Implemented the honest-param + docs fix in `Source/PinWright/Private/Handlers/Level/LevelStructureHandler.cpp` (`level.structure.add_level_blueprint_node`, ~line 1707): (1) `nodeName` is now applied to the created node as its `NodeComment` label instead of being dropped, and **echoed back under `nodeName`**; the auto-generated node title moved to a distinct `nodeTitle` response key (the two no longer share one key). (2) Position now accepts the sibling flat `x`/`y` spelling as an alias for `nodePosition.{x,y}` — both are declared in the param spec (so the dispatcher's `ValidateHandlerParams` no longer rejects flat x/y as `UNKNOWN_PARAMS`) and read with the object shape taking precedence. (3) The registration summary and a new `### level.structure.add_level_blueprint_node` section in `Docs/wiki-src/level.structure.md` now state these verbs create UNBOUND stub nodes only and steer callers to `blueprint.graph.create_node` for bound authoring; the response also carries a `note` to the same effect. Regression test `PinWright.level.structure.add_level_blueprint_node.HonorsNodeNameAndFlatPos` in `Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp` drives the production handler against the live editor world's level-script Blueprint and asserts: response `nodeName` echoes the caller's value (not the auto title), a distinct `nodeTitle` carries the auto title, flat `x`/`y` land at the requested coords, and the created node carries `nodeName` as its `NodeComment`; plus spec assertions that flat `x`/`y` are declared (guards the alias path even in the commandlet skip branch). The node is removed and the level dirty flag restored on exit. Did NOT add event/function binding (out of scope per the reword). Companion bare-`assetPath` trap remains tracked in `E-open-level-blueprint-unusable-assetpath`.
- `#2-additional-nodeposition-ignored` `OPEN` reporter — Re-confirmed live on `ExampleProjectWelcome` (seed `level.structure.add_level_blueprint_node`). `call("level.structure.add_level_blueprint_node", {"nodeClass":"K2Node_Event","nodeName":"MyCustomBeginPlayName","nodePosition":[300,300]})` -> ok `{"nodeClass":"K2Node_Event","nodeName":"Event None","posX":0,"posY":0,"nodeCreated":true}`: confirms (1) `nodeName` dropped + key reused (passed `"MyCustomBeginPlayName"`, response echoes the auto title `"Event None"`) and (2) unbound `UK2Node_Event` (`Event None`, no binding). NEW symptom of the same shim: **`nodePosition` is also ignored** — passed `[300,300]`, response reports `posX:0,posY:0` (the requested coordinates are silently dropped just like `nodeName`). Also note the param-shape footgun: `x`/`y` (the `blueprint.graph.create_node` spelling) are rejected `[UNKNOWN_PARAMS] … Valid parameters: [nodeClass, nodeName, nodePosition]` — this verb wants a single `nodePosition` array, divergent from the sibling graph verb. Companion handle-path trap (`open_level_blueprint` bare `assetPath` -> `blueprint.graph.* [ASSET_NOT_FOUND]`) also re-confirmed live this run, tracked in `E-open-level-blueprint-unusable-assetpath`.
- `#1-initial-audit` `OPEN` reporter — Source-confirmed in `LevelStructureHandler.cpp`: `add_level_blueprint_node` (registered line 1706) declares `RPC_PARAM_OPT("nodeName")` (1710), reads it to local `NodeName` (1717), but never applies it — node is `NewObject<UK2Node>(EventGraph, NodeClassObj)` (1785) + `AllocateDefaultPins()`, and the response echoes the auto-generated node title back under key `nodeName` (1822). No event/function-ref binding param exists, so events come out as unbound `UK2Node_Event` (no `EventReference`) and calls have no target UFunction. The dedicated `level.structure.*` level-BP authoring sub-family therefore cannot produce a bound BeginPlay->PrintString; live attempt agent abandoned it for `blueprint.graph.create_node` (with `eventName`) on the level-script object path, learning the gap only by reading the handler C++. `docs/wiki-src/level.structure.md` does not name these verbs or warn of the limitation. Distinct from `E-open-level-blueprint-unusable-assetpath` (that ticket = the returned bare `assetPath` not round-tripping into `blueprint.graph.*`); this ticket = the dedicated authoring verbs being non-functional shims + the silently-dropped/echo-shadowed `nodeName` param.
