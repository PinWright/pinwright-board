---
id: F-batch-connect-pins
title: "No batch `blueprint.graph.connect_pins`: wiring N links costs N calls, N undo entries and a separate compile"
status: IN-REVIEW
severity: Medium
category: feature
tags: [blueprint, graph, batching, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# No batch `blueprint.graph.connect_pins`

`blueprint.graph.connect_pins` wires one output pin to one input pin per call. Hand-wiring a graph
of any size (the fixup path after `blueprint.graph.create_node`) therefore costs one round trip per
link, one undo entry per link, and a separate `blueprint.compile` call, because `connect_pins` never
compiles. Round-trip count is the cost `docs/rpc-design.md` §8 says to minimise. The sibling
`blueprint.graph.set_pin_default_values` already batches pin defaults; linking has no equivalent.

**Fix:** `blueprint.graph.connect_pins_batch` takes `links: [{fromNodeId, fromPinName, toNodeId,
toPinName}, ...]` under one `FScopedTransaction`, reports every entry in `results[]` with
`successCount`/`failureCount`, refuses an empty array, and compiles once when `compile: true`
through `CompileBlueprintWithDiagnostics` behind the `BlueprintReinstancingGuard` consent gate.

## History
- `#1-one-call-per-link` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis (plan task 7). Wiring N pins takes N `connect_pins` calls plus a `blueprint.compile`; no batch verb exists. Severity Medium: doable, but only through many extra calls.
- `#2-batch-verb-added` `IN-REVIEW` developer - Added `blueprint.graph.connect_pins_batch` in `Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphConnectionsHandler.cpp`. Params: `assetPath` (alias `path`), `links` (required, non-empty; `INVALID_ARGUMENT` otherwise), `graphName`, `compile` (default false), `allowReinstancing`. One transaction; each entry independent, never rolled back by a sibling failure. Per entry: `index`, the four echoed fields, `success`, `connection` (`direct` / `alreadyConnected` / `conversionNode` / `promotion`, plus `brokeExistingLinks` when the schema replaced a link) or `error` + `message` (`INVALID_ARGUMENT`, `NODE_NOT_FOUND`, `PIN_NOT_FOUND` with `sourcePinLookup`/`targetPinLookup`, `CONNECTION_FAILED` carrying the schema's reason). Already-present links are reported, not re-made, so a retry converges. Success is re-measured on the final graph: an entry whose link a later entry replaced on a single-link pin is flipped to new code `LINK_SUPERSEDED` (`Handlers/ErrorCodes.h`, catalogued in `docs/error-code-catalog.md`). `compile:true` refuses with `LIVE_INSTANCES_WOULD_BE_REINSTANCED` before any link is made, else compiles once after the transaction and reports `compiled`/`status`/`compileErrors`/`compileWarnings`; without it `compiled:false`. Verb added to the tick-unsafe table (`Dispatch/SafePoint.cpp`) and to the compile-verb inventory and consent sites in `Tests/Infra/TestHandlerTickSafetyRatchet.cpp` (inventory 59 -> 60). The file now adopts `ErrorCodes::ERR_*`, so `connect_pins`/`break_pin_links` literals were converted too. Tests in `Tests/Blueprint/TestBlueprintConnectPinsBatch.cpp`: `PinWright.blueprint.graph.connect_pins_batch.{WiresEveryLink, PartialFailureKeepsSuccesses, LaterEntrySupersedesEarlier, RetryReportsAlreadyConnected, CompileFlagCompiles, EmptyLinksRejected}`. Wiki: `docs/wiki-src/blueprint.graph.md` gains `connect_pins` (never compiles) and `connect_pins_batch` sections. Not compiled and suite not run (another build held the machine).
