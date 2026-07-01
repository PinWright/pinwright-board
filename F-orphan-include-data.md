---
id: F-orphan-include-data
title: "`find_orphaned_nodes` default should include data-only nodes"
status: DONE
severity: ""
category: feature
tags: []
---

# `find_orphaned_nodes` default should include data-only nodes

Default mode misses disconnected pure nodes. Multiple "clean" checks missed 5-8 stale nodes found later with `includeDataOnly: true`.

**Proposal:** Default `includeDataOnly` to `true`, or report count even when not deleting.

## History
- `#1-missed-pure-orphans` `OPEN` reporter — Ran find_orphaned_nodes 3 times thinking graph was clean. Later found 8 pure orphans with includeDataOnly:true.
- `#2-changed-default-to-true` `IN-REVIEW` developer — Changed `RPC_PARAM_DEF` default from `"false"` to `"true"`. Simplified both runtime resolution paths to `GetJsonBoolField(Payload, ..., true)`. Updated Test 5 (renamed to PureNodeIncludedByDefault, expects orphanedCount >= 1). Added new opt-out test (PureNodeExcludedWhenOptOut, explicit false → orphanedCount == 0).
- `#3-verified-default-true` `DONE` tester — Verified: created disconnected MakeVector2D pure node on W_McpTestTemp. Default find_orphaned_nodes found it (orphanedCount: 1, hasExecPins: false). Explicit includeDataOnly: false returned orphanedCount: 0. Default is now true.
