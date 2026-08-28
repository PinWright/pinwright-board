---
id: B-niagara-connect-pins-sends-no-graph-notification
title: "niagara.graph.connect_pins mutates LinkedTo and returns without any graph notification, so an open editor's pin caches and splines go stale"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, graph, connect_pins, notification, stale-cache, OverridePinCache, slate]
encounters: 1
lastSeen: 2026-08-28
---

# The same missing notification just fixed on two other verbs

`niagara.graph.connect_pins` mutates `LinkedTo` and returns. It sends no graph notification at all, so
an open Niagara editor keeps `UNiagaraStackFunctionInput::OverridePinCache` (a bare `UEdGraphPin*`)
and `SGraphPanel`'s connection splines pointing at the pre-edit state.

This is the same defect `B-niagara-editor-open-guard-missing-on-mutators` fixed on
`reset_module_input` and `clear_module_overrides` — and the same one-line fix: end at
`NotifyNiagaraGraphChanged`, whose bare `NotifyGraphChanged()` triggers
`SGraphPanel::PurgeVisualRepresentation()` and clears the cache.

Correctness rather than a crash: nothing is freed here, so the stale cache misreports rather than
dangles. It still means the editor shows a wiring that is not what the graph holds.

## History
- `#1-named-by-the-notification-fix` `OPEN` reporter — Named by the agent that added
  `NotifyNiagaraGraphChanged` to the two override-pin verbs, having audited every mutator in the four
  Niagara handler files for the same shape. Source reading, not reproduced.
- `#2-notify-graph-changed-on-connect` `IN-REVIEW` developer — "Ended the successful
  `niagara.graph.connect_pins` branch in `NiagaraGraphHandler.cpp` with the same bare
  `TargetGraph->NotifyGraphChanged()` the file's `create_node` verb already uses, so
  `OnGraphChanged` reaches `SGraphPanel::PurgeVisualRepresentation` and the stack's
  `OverridePinCache`. Confirmed in engine source that `UEdGraphSchema_Niagara::TryCreateConnection`
  reaches only `UNiagaraNode::PinConnectionListChanged`, whose `NotifyGraphNeedsRecompile`
  broadcasts `OnGraphNeedsRecompile` and returns before `Super::NotifyGraphChanged`. Regression
  test `PinWright.niagara.graph.connect_pins.NotifiesGraphChanged` in `TestNiagaraHandlers.cpp`."
