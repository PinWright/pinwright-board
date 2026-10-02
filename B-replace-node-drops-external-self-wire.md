---
id: B-replace-node-drops-external-self-wire
title: "blueprint.graph.replace_node on an external-owner VariableGet silently drops the wired self link (connectionsRewired 0, connectionsDropped [])"
status: DONE
severity: Medium
category: bug
tags: [blueprint, blueprint-graph, replace_node, variableget, self-pin, external-owner, silent-drop]
encounters: 1
lastSeen: 2026-09-29T14:40:00Z
---

# replace_node drops a wired self pin without reporting it

The old node was a `Get PlayerIndex` on an **external** owner (`OpponentInfo`), with its `self` pin
wired to a `Get OpponentInfo` variable node. Replacing it with a getter for another member of the same
owner:

```
blueprint.graph.break_pin_links {nodeId:"9334DA9C...", pinName:"PlayerIndex"}      # output wire removed on purpose
blueprint.graph.replace_node {assetPath:"/App/App/UI/LobbyAndMenu/Elements/W_OnlineUsersListItem",
  graphName:"EventGraph", nodeId:"9334DA9C41B6D55BEAEC43B4689DF100",
  newNodeType:"VariableGet", target:"OpponentInfo::PlayerState"}
-> {"newNodeId":"01A0ED851D057CD7BDD1B9CB857DA891","connectionsRewired":0,"connectionsDropped":[], ...}
```

`get_pin_details_batch` on the new node: `self` (input, `OpponentInfo`) has **no** `linkedTo`. The old
`self <- 8F77779F...:OpponentInfo` wire is gone, and the response lists nothing under
`connectionsDropped`. Both nodes have a `self` pin of the same type, so the wire should have moved. For
an external-owner accessor an unwired `self` is a compile error ("uses an invalid target"), so the
caller has to spot the loss on their own.

This looks like the "skip self when replacing variable accessors" rule from `F-bp-graph-replace-node-rpc`
applied to an external-owner node, where `self` carries the object and must be kept.
`B-replace-node-variableget-loses-self-context` (IN-REVIEW) covers the self-member case; this is the
external-owner counterpart, and it at least needs to be reported.

**Workaround:** `connect_pins` the owner object back into the new node's `self`.

**Fix (proposed):** move the `self` link like any other matched pin when the replaced accessor's `self`
is wired (external context). If a pin is deliberately skipped, list it under `connectionsDropped`.

## History
- `#1-external-self-wire-dropped` `OPEN` reporter - Filed from the PDS QA #744 fix, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`. Spotted by reading pins back after the replace; fixed by hand with one `connect_pins`. Cheap, but the response reported full success.
- `#2-wired-self-moves` `IN-REVIEW` developer - The non-CallFunction self skip in `replace_node` pin migration now applies only to an **unwired** `self`. A wired `self` is matched like any pin (exact name, but never onto a hidden self-context `self`), moved with `MovePinLinks` and counted in `connectionsRewired`; when no compatible visible `self` exists it is recorded in `connectionsDropped` with reason `NO_MATCH` (or an orphan placeholder under `allowOrphanPlaceholders`) instead of refusing the replace. Files: `Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphCrudHandler.cpp`, `Source/PinWright/Private/Tests/Blueprint/TestBlueprintReplaceNode.cpp`, `docs/wiki-src/blueprint.graph.md` (replace_node "The `self` pin" paragraph), `CHANGELOG.md`. Tests: `PinWright.blueprint.graph.replace_node.VariableGet_ExternalOwner_KeepsSelfWire` (Pawn::BaseEyeHeight -> Pawn::AIControllerClass, self wire kept), `PinWright.blueprint.graph.replace_node.VariableGet_ExternalOwner_IncompatibleSelfReported` (Actor::Tags with Actor ref -> Pawn::BaseEyeHeight, self in connectionsDropped NO_MATCH).
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.blueprint.graph.replace_node.VariableGet_ExternalOwner_KeepsSelfWire` (a wired external `self` moves to the new getter and counts in `connectionsRewired`) and `PinWright.blueprint.graph.replace_node.VariableGet_ExternalOwner_IncompatibleSelfReported` (no compatible self -> listed in `connectionsDropped` with `NO_MATCH`). Both halves of the proposed fix are met. Limit: fixtures use engine classes (Pawn, Actor), not the PDS `OpponentInfo` owner.
