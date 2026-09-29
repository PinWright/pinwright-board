---
id: B-node-details-omit-pin-links
title: "blueprint.graph.get_node_details(_batch) omit pin links: a wired input pin reads as its unused defaultValue (\"0\") with no linkedTo"
status: OPEN
severity: Medium
category: bug
tags: [blueprint, blueprint.graph, get-node-details, get-node-details-batch, pins, linkedTo, readback, misleading]
encounters: 1
lastSeen: 2026-09-29T13:05:00Z
---

# Node details show a connected pin as if it held its default

`blueprint.graph.get_node_details` and `get_node_details_batch` on
`/App/App/UI/LobbyAndMenu/Elements/W_OnlineUsersListItem`, `EventGraph`, node
`5DD2BC524F7324F8FB4D12964AB40FA5` (`GetDronePlayerControllerPure`) returned the `PlayerIndex` input
as `{"pinName":"PlayerIndex","direction":"Input","pinType":"int","defaultValue":"0"}` and nothing
else. The pin is wired: `blueprint.graph.get_pin_details` on the same pin returns
`linkedTo: ["9334DA9C41B6D55BEAEC43B4689DF100:PlayerIndex"]` (a `Get PlayerIndex` from the
`OpponentInfo` variable). Reading only the node-details payload, the call looks like it always
targets player 0, which is a wrong conclusion about the graph's behaviour.

**Workaround:** follow up every pin that matters with `get_pin_details`, or read
`get_execution_flow` `dataInputs`.

**Fix (proposed):** include `linkedTo` (node id + pin name) on every pin in both methods, and omit
or flag `defaultValue` when the pin is connected (the engine ignores it then).

## History
- `#1-playerindex-reads-as-zero` `OPEN` reporter - Filed from a PDS multiplayer investigation on UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin `61c243f5`. Almost concluded the room-admin row buttons always target player 0; `get_pin_details` showed the link. Cheap (one extra call), but a silent misread risk.
