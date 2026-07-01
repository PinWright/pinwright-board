---
id: B-node-layout-poor
title: "Node auto-formatting produces poor layouts"
status: DONE
severity: Medium
category: bug
tags: []
---

# Node auto-formatting produces poor layouts

After `compile_bpir` or `insert_bpir_at_node`, nodes overlap, long horizontal chains extend off-screen, no vertical grouping by logical block. Manually-built graphs are clean and readable; BPIR-generated graphs are not.

## History
- `#1-initial-repro` `OPEN` reporter — All 5 photo inspection widgets have messy layouts compared to hand-built sphere hunt/gas leak.
- `#2-layout-improvements` `IN-REVIEW` developer — Node layout improvements already implemented; moved to review.
- `#3-verified-layout-clean` `DONE` tester — Verified: compiled 4-node BPIR on W_McpVerifyTemp (Event Construct → GetName → PrintString → SetVisibility). Node positions: entry (0,1312), pure (32,1440), impure chain (496,1440), (752,1440). No overlap, ~256px horizontal spacing, pure nodes offset vertically. Layout is clean.
