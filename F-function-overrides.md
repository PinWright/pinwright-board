---
id: F-function-overrides
title: "No way to create function overrides via any MCP tool"
status: DONE
severity: ""
category: feature
tags: []
---

# No way to create function overrides via any MCP tool

Neither `blueprint_add_function` nor `compile_bpir` can create BlueprintNativeEvent overrides. Related to B-native-event-override.

**Proposal:** `blueprint_add_function` with `override: true`, or `compile_bpir` auto-detecting parent declarations.

## History
- `#1-override-impossible-mcp` `OPEN` reporter — GetContentPanel override on route widget impossible via MCP. Manual editor work required.
- `#2-fixed-via-native-event` `IN-REVIEW` developer — Fixed via B-native-event-override fix. `compile_bpir` now auto-detects parent native events via ParentClass fallback + skeleton-ensure. Both `entry function X()` (auto-detect) and `entry override X()` (explicit) now work for BlueprintNativeEvent functions.
- `#3-verified-override-compile` `DONE` tester — Verified via B-native-event-override test. `entry override GetContentPanel()` on W_McpOverrideTest (TrackHUDLayout child) compiled clean.
