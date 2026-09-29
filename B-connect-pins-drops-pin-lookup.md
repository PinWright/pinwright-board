---
id: B-connect-pins-drops-pin-lookup
title: "blueprint.graph.connect_pins builds the live-pin lookup on PIN_NOT_FOUND and then sends the error without it"
status: IN-REVIEW
severity: Medium
category: bug
tags: [blueprint, graph, connect-pins, error-data, pin-lookup, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T00:00:00Z
---

# connect_pins PIN_NOT_FOUND discards its pin lookup

`blueprint.graph.connect_pins` (`Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphConnectionsHandler.cpp`),
when either pin does not resolve, builds a `Result` object carrying `sourcePinLookup` /
`targetPinLookup` (`BuildPinLookupPayload`: `availablePins`, `inputPins`, `outputPins`,
`closestMatches`) and then calls the two-argument `Ctx.SendError(ERR_PIN_NOT_FOUND, ...)`, so the
object is dropped. The caller gets a bare "Could not find source or target pin." while the
`blueprint.graph` wiki's error table says `PIN_NOT_FOUND` lists the node's live pins. A caller
following the docs finds no pin list and needs an extra `blueprint.graph` inspection call per miss.
The sibling `blueprint.graph.connect_pins_batch` in the same file already returns the lookups on
its failed item.

**Fix:** pass the built object as error data (`SendError(Code, Message, Result)`), same keys as the
batch item.

## History
- `#1-lookup-discarded` `OPEN` reporter - Found in the follow-up review of the 2026-09-28 gap-analysis wave: `connect_pins` PIN_NOT_FOUND branch builds `sourcePinLookup`/`targetPinLookup` into a local `Result` and sends the error without it; `docs/wiki-src/blueprint.graph.md` promises the live pins. Severity Medium: a documented readback is missing and forces a fallback inspection call.
- `#2-lookup-in-error-data` `IN-REVIEW` developer - `BlueprintGraphConnectionsHandler.cpp`: the PIN_NOT_FOUND branch now calls `Ctx.SendError(ERR_PIN_NOT_FOUND, ..., Result)` with `sourcePinLookup` / `targetPinLookup` (only for the pin that missed), the same shape `connect_pins_batch` puts on a failed item; the stray `error` string that was never sent is gone. Test `PinWright.blueprint.graph.connect_pins.PinNotFoundCarriesLivePins` (`Tests/Blueprint/TestBlueprintConnectPinsBatch.cpp`): unknown source pin returns PIN_NOT_FOUND whose data has `sourcePinLookup.availablePins` containing `then`, no `targetPinLookup`, and no link made. Not compiled or run yet (wave build pending).
