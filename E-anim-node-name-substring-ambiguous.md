---
id: E-anim-node-name-substring-ambiguous
title: "AnimGraph nodeName resolution is first-match substring with no ambiguity detection; success echoes the raw substring, not the resolved node"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [animation, anim-graph, node-resolution, set_sync_group, bind_player_asset, ambiguous]
---

# AnimGraph `nodeName` silently resolves the first substring match and misreports it

`animation.authoring.set_sync_group`, `bind_player_asset`, and any other
AnimGraph mutator that takes a `nodeName` resolve the target node through
`AnimGraphConstructionUtils::FindAnimGraphNodeByTitleSubstring`
(`AnimGraphConstructionUtils.cpp:262`), which loops the graph's nodes and
returns the **first** one whose `GetNodeTitle(ENodeTitleType::ListView)`
**`.Contains(nodeName)`** — a substring test with **no ambiguity detection
and no exact-match preference**. When two or more nodes' titles contain the
given substring, it silently mutates an arbitrary one (graph traversal
order) and reports `success: true`.

This is concretely misleading on two counts:

1. **Silent wrong-node mutation.** A `nodeName` substring that matches
   several nodes picks one with no warning. The very common case is two
   stock player nodes — both titled `Sequence Player '<asset>'` — where a
   caller who passes the shared prefix `"Sequence Player"` hits one of them
   blind. (Two *unbound* `SequencePlayer` nodes are both titled exactly
   `Sequence Player`, so they are literally indistinguishable by this
   resolver.)
2. **The result misreports the target.** The response echoes back
   `"nodeName"` as the *input substring verbatim*, not the actual resolved
   node's title. The caller cannot tell from the response which node was
   mutated, so the readback gives false confidence that the named node was
   the one touched.

The `nodeName` parameter doc only says "Name of the asset-player AnimGraph
node to mutate" and `set_sync_group`'s notes say it resolves "with the same
`nodeName` convention as `bind_player_asset`" — but the convention
(title-substring `.Contains`, first match wins) is never documented, so a
caller cannot know how unique the substring must be or that ambiguity is
swallowed rather than reported.

This sits next to existing first-match hazards filed on other surfaces
(`B-set-widget-text-hits-wrong-instance`, `E-remove-event-multi-match-dedup`,
`B-bpir-asyncaction-disambiguation` — the last of which was fixed by adding
an ambiguity error listing candidates); the AnimGraph node resolver has the
same shape and no such guard.

**Verbatim repro** (replayed live against `mcp__editor-automation__call`):

Setup — two SequencePlayer nodes in `AnimGraph`, bound to distinct assets, so
their list-view titles are `Sequence Player 'Dino_Idle'` and
`Sequence Player 'Dino_Walk'` (both contain the substring `Sequence Player`):

```
animation.authoring.add_graph_node {blueprintPath:".../ABP_DinoLoco_Replay",
  nodeClass:"AnimGraphNode_SequencePlayer", x:-400, y:-100,
  bindAsset:".../Dino_Idle.Dino_Idle"}
  -> {"nodeName":"Sequence Player 'Dino_Idle'", ... "success":true}
animation.authoring.add_graph_node {..., x:-400, y:200,
  bindAsset:".../Dino_Walk.Dino_Walk"}
  -> {"nodeName":"Sequence Player 'Dino_Walk'", ... "success":true}
```

Ambiguous mutate — `nodeName:"Sequence Player"` matches BOTH nodes, yet:

```
animation.authoring.set_sync_group {blueprintPath:".../ABP_DinoLoco_Replay",
  nodeName:"Sequence Player", groupName:"Locomotion", role:"AlwaysFollower"}
  -> {"nodeName":"Sequence Player","groupName":"Locomotion",
      "role":"AlwaysFollower","syncMethod":"SyncGroup","success":true}
```

No `AMBIGUOUS_NODE` error, no candidate list, no warning — and the echoed
`"nodeName":"Sequence Player"` does not say which of the two nodes was
actually mutated. (A non-matching name does fail cleanly:
`nodeName:"IdlePlayer"` -> `[NODE_NOT_FOUND] Node 'IdlePlayer' not found in
AnimGraph` — so the miss path is fine; only the *multi-match* path is the
problem.)

**Workaround:** Pass a `nodeName` substring unique to one node — for stock
player nodes, the bound asset name (e.g. `"Dino_Idle"`) rather than the
shared `"Sequence Player"` prefix. This only works if every target node is
bound to a distinctly-named asset; unbound players cannot be disambiguated
at all.

**Fix:** Make `FindAnimGraphNodeByTitleSubstring` (or its callers) collect
**all** title-substring matches and (a) prefer an exact title match if one
exists, else (b) when >1 node matches, fail with an `AMBIGUOUS_NODE` error
that lists the candidate titles — mirroring the two-pass exact-then-fuzzy
disambiguation already shipped for `B-bpir-asyncaction-disambiguation`. At
minimum, echo the **resolved node's actual title** in the response
`nodeName` instead of the input substring, so callers can verify the hit.

## History
- `#1-initial-repro` `OPEN` reporter — Seed `animation.authoring.set_sync_group`; the verb itself is correct (set Walk=AlwaysLeader / Idle=AlwaysFollower, Standalone clears group + resets role to `CanBeLeader` + `DoNotSync`, all readbacks accurate — matches the wiki). The finding is in the shared node resolver: `FindAnimGraphNodeByTitleSubstring` (`AnimGraphConstructionUtils.cpp:262`) returns the FIRST `GetNodeTitle(ListView).Contains(nodeName)` match with no ambiguity guard. Replay-confirmed: with two SequencePlayer nodes titled `Sequence Player 'Dino_Idle'` and `Sequence Player 'Dino_Walk'`, `set_sync_group nodeName:"Sequence Player"` (matches both) returned `success:true` with `"nodeName":"Sequence Player"` echoed verbatim — silently mutating one node, no candidate list, no indication of which. Clean miss path (`nodeName:"IdlePlayer"` -> `[NODE_NOT_FOUND]`). No existing board ticket covers AnimGraph nodeName ambiguity (`F-anim-sync-group-on-player-nodes` is the DONE typed-setter feature; the first-match siblings on widget/remove_event/bpir-async surfaces are separate handlers).
- `#2-fix` `IN-REVIEW` developer — Added a two-pass exact-then-substring resolver `AnimGraphConstructionUtils::ResolveAnimGraphNodeByTitle` (returns `FAnimNodeResolveResult{Status: Found/NotFound/Ambiguous, Node, Candidates}`) in `Source/.../Handlers/Animation/AnimGraphConstructionUtils.{h,cpp}`, mirroring the `B-bpir-asyncaction-disambiguation` precedent: a single exact-title match wins outright; else a single substring match wins; else >1 substring match → Ambiguous with every candidate title listed. Added a file-local `ResolveAnimNodeOrSendError` helper in `AnimationAuthoringHandler_AnimBlueprint.cpp` that sends `AMBIGUOUS_NODE` (candidate-listing message, house style per LiveUiSnapshot/GameplayTags) or `NODE_NOT_FOUND`, and hands back the resolved node's ACTUAL list-view title. Rewired all 5 callers (`set_sync_group` :2271, `bind_player_asset` :2119, `set_anim_graph_pin_exposed` :1932, `set_anim_graph_pins_exposed` :2006, `add/set_layered_blend` :1663 — the last keeps its WRONG_NODE_TYPE distinction) to use it, and changed each response to echo the resolved title instead of the raw input substring (fixes the verbatim-echo misreport). The old first-match `FindAnimGraphNodeByTitleSubstring` is retained as documented-legacy API with no remaining callers. Regression test `FAnimAuthoringNodeNameAmbiguityTest` (`Tests/Assets/TestAnimGraphHandlers.cpp`, "EditorAutomationRpcGateway.anim.authoring.NodeNameAmbiguity") authors two identically-titled `Sequence Player` nodes and asserts: the resolver reports Ambiguous with 2 candidates; `set_sync_group` with the shared substring fails `AMBIGUOUS_NODE` (not silent success) and leaves both nodes' sync groups untouched; then with a single bound node, the response echoes the resolved full title `Sequence Player '<asset>'`, not the input substring. Reverting the resolver or the handler wiring fails it. Not compiled here (later phase compiles/tests).
