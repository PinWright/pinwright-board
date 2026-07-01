---
id: B-replace-orphan-body
title: "`compile_bpir` replace mode doesn't clean old body subgraph"
status: DONE
severity: Medium
category: bug
tags: []
---

# `compile_bpir` replace mode doesn't clean old body subgraph

Replace mode with matching signature reuses entry but creates new body alongside old body. Old body becomes orphaned, doubling graph size.

## History
- `#1-replace-created-orphan-nodes` `OPEN` reporter — OnDroneArmedEvent replace created 20 new nodes alongside 13 old ones. Required find_orphaned_nodes cleanup.
- `#2-verified-replace-deleted-old-body` `DONE` tester — Verified: replace-mode recompile of OnDroneArmedEvent deleted old exec body nodes. Data-only pure orphans from old body still need includeDataOnly cleanup (tracked as F-orphan-include-data), but exec chain properly replaced.
