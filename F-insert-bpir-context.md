---
id: F-insert-bpir-context
title: "`insert_bpir_at_node` missing `context` parameter from MCP schema"
status: DONE
severity: ""
category: feature
tags: []
---

# `insert_bpir_at_node` missing `context` parameter from MCP schema

The BPIR insertion API docs (`bpir-insertion-api.md`) describe a `context` object parameter for injecting external pin references (`$VarName` → `"nodeGuid:pinName"`), but the MCP tool schema doesn't expose it (`additionalProperties: false` rejects unknown params). This forces re-deriving values (e.g., re-casting to PhotoInspectionTrack) instead of referencing existing node outputs, creating redundant nodes.

**Proposal:** Add `context` as an optional object parameter to the MCP schema for both `insert_bpir_at_node` and `insert_bpir_before_node`.

## History
- `#1-missing-context-param` `OPEN` reporter — Had to add 3 redundant nodes (GetDroneGameStatePure → GetTrackPure → cast) in W_HUD_PhotoInspection Tick because couldn't reference existing cast result.
- `#2-param-already-existed` `IN-REVIEW` developer — Verified in code: `context` param already exists in both `insert_bpir_at_node` (RPC_PARAM_OPT) and `insert_bpir_before_node`. `ResolveContextInjections()` helper resolves `nodeGuid:pinName` references and calls `Compiler.InjectExternalVariable()`. No code change needed — board status was stale.
- `#3-verified-context-injection` `DONE` tester — Verified: insert_bpir_at_node with context:{"ExistingName":"<GetName-nodeId>:ReturnValue"} and code `call PrintString(InString: $ExistingName)`. New PrintString.InString pin linked to GetName:ReturnValue. Context injection works end-to-end.
