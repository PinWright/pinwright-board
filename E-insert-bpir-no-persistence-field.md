---
id: E-insert-bpir-no-persistence-field
title: "blueprint.insert_bpir_at_node / insert_bpir_before_node success responses carry no persistence block, unlike compile_bpir after E-compile-bpir-no-persistence-field"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, insert_bpir, save, persistence, pendingFlush, response-shape]
encounters: 1
lastSeen: 2026-10-02T00:00:00Z
rice: [1, 2, 1, 1]
priority: 17
---

# The two insert verbs are still silent about durability

`blueprint.compile_bpir` now reports `saveRequested` / `markedForSave` / `saved` / `pendingFlush`
on every kept placement (`E-compile-bpir-no-persistence-field` `#2-persistence-block`, via
`McpSafeAssetSave` + `AddMarkDirtySaveReport` after `SendBpirPendingResponse` is prepared in
`Source/PinWright/Private/Handlers/Blueprint/BpirCompilerHandler.cpp`). Its two siblings in the
same file do not: `blueprint.insert_bpir_at_node` (success branch around `BpirCompilerHandler.cpp:626`)
and `blueprint.insert_bpir_before_node` (around `:831`) still answer `compiled:true, success:true`
with no save field, although neither writes the `.uasset`.

Same trap as the parent ticket, smaller reach (the insert verbs are used less than
`compile_bpir`): a caller reads durable success, an editor crash before `asset.save` loses the
inserted nodes.

## Fix

Apply the same two lines on both success paths (mark dirty through `McpSafeAssetSave`, then
`AddMarkDirtySaveReport(Payload, BP, true)`), add a handler-level test per verb mirroring
`PinWright.blueprint.compile_bpir.ReportsUnsavedPersistence`, and say "does not save" on both
wiki sections.

**Workaround:** `asset.save` after every insert.

## History
- `#1-filed` `OPEN` developer — Found by source while fixing `E-compile-bpir-no-persistence-field`; an independent review of that change flagged the two insert verbs as the remaining gap. Source-only: no editor or MCP call run. Severity Low: same friction as the parent, lower-reach verbs.
