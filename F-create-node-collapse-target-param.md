---
id: F-create-node-collapse-target-param
title: "Collapse create_node's class-specific name params into a single `target` field (vocabulary parity with replace_node)"
status: DONE
severity: Medium
category: feature
tags: [blueprint-graph, mcp-rpc, api-vocabulary, consistency]
---

# Vocabulary split between `create_node` and `replace_node`

`blueprint.graph.replace_node` (v1.1) takes a single unified `target` string
that accepts `"Foo"` (class inferred), `"Class::Foo"`, or `"Class.Foo"` —
interpreted per `newNodeType`. `blueprint.graph.create_node` still takes the
old verbose param set: `memberName`, `memberClass`, `variableName`,
`eventName`, `targetClass`. Same vocabulary universe, different field names
per verb. The handler file's own comment flags this as deliberate coupling
("Mirrors the create_node factory pattern — keep these branches in
lockstep"); now that `replace_node` has diverged, lockstep has been broken
in exactly the way the comment warned about.

For callers writing automation scripts that mix create + replace operations
(BPIR-style refactor passes, codegen flows), this means remembering
"`create_node` wants `variableName`, but `replace_node` wants `target`"
per verb. Same idea, two field names. Forward-only friction.

**Proposed fix:** add optional `target` to `blueprint.graph.create_node` for
the explicit factory branches that overlap `replace_node`: `VariableGet`,
`VariableSet`, `CallFunction`, `Event`, `CustomEvent`, and `Cast` /
`CastTo`. `target` accepts bare `Foo`, qualified `Class::Foo` /
`Class.Foo`, or a class name for casts. Keep the existing aliases
(`variableName`, `memberName` + `memberClass`, `eventName`, `targetClass`)
working when `target` is absent; alias requests must not fail solely because
they use legacy vocabulary. Move only the shared target-string parser and
small reusable variable/function resolution helpers into
`BlueprintHandlerUtils`, and reuse the shared parser in `replace_node` without
broad factory abstraction or deprecation policy.

**Out of scope:** generic-K2Node-subclass fallback for plugin custom nodes
and engine nodes outside the explicit factory branches (Timeline, FormatText,
GetArrayItem, AddDelegate, etc.) — separate ticket.

**Files affected:**
- `Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGateway\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp` (create_node factory ~1031-1259)
- `Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGateway\Private\Handlers\Blueprint\BlueprintHandlerUtils.h/.cpp` (shared target-parser helpers)
- `Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGatewayTests\Private\Blueprint\TestBlueprintHandlers.cpp` (target and legacy alias regression tests)
- `Plugins\EditorAutomationRpcGateway\docs\wiki\blueprint.graph.md` (create_node section + JSON examples)

## History
- `#1-initial-feature-request` `OPEN` reporter — `replace_node` v1.1 (`F-bp-graph-replace-node-rpc` IN-REVIEW) collapsed nine class-specific name params into a unified `target` field. `create_node` still uses the old verbose vocabulary, breaking the in-file "keep these branches in lockstep" coupling. Proposal: collapse `create_node` the same way, keep legacy param names as back-compat aliases for one release, lift the parser into `BlueprintHandlerUtils` so both verbs share resolution.
- `#2-create-node-target-param` `IN-REVIEW` developer — Changed `BlueprintGraphHandler.cpp`, `BlueprintHandlerUtils.h/.cpp`, `TestBlueprintHandlers.cpp`, and `docs/wiki/blueprint.graph.md` to add `create_node.target` support for variable/function/event/custom-event/cast branches, preserve legacy aliases, reuse shared target parsing/resolution helpers, document the new surface, and add target plus legacy CallFunction regression tests.
- `#3-verify-target-param` `DONE` tester — Verified: `blueprint.graph.create_node` wiki schema lists `target` as optional with the documented format. Live call with `nodeType=CallFunction, target="KismetSystemLibrary::PrintString"` on a temp BP returned `nodeName=K2Node_CallFunction_0` (success). Temp BP cleaned up via `asset.delete`.
