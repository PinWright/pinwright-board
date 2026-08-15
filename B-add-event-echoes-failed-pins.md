---
id: B-add-event-echoes-failed-pins
title: "blueprint.add_event echoes failed pins as created, and blueprint.get corroborates it from the registry"
status: IN-REVIEW
severity: High
category: bug
tags: [blueprint, silent-false-success, readback, pins]
encounters: 1
---

# blueprint.add_event echoes failed pins as created

At `b92ba268`, `blueprint.add_event` collected every pin it failed to create into a block-local
`FailedParams` (`Handlers/Blueprint/BlueprintEventHandler.cpp:192`, `:215`, `:231`), wrote it to a
`UE_LOG` at `:237-238`, and let it go out of scope at `:240`. The response then set
`success: true` unconditionally (`:365`) and emitted the caller's **request** array as
`parameters` (`:370`, sourced from `:96-101`) — an array with no dependency whatsoever on pin
creation having worked. The agent's documented next step is `connect_pins` against a pin that
does not exist.

Worse, the most common bad input was not even counted as a failure. `AddUserDefinedPin`
(`BlueprintHandlerUtils.cpp:1195-1202`) falls back to a `PC_Wildcard` pin when the type spec does
not resolve, and returns **true**, so an unresolvable type produced a malformed pin and a clean
success. The honest sibling for exactly this already existed and was unused here:
`RejectWildcardPinParams` / `FindFirstPinParamWildcardFallback`
(`BlueprintHandlerUtils.cpp:1379-1396`, `:1345-1377`), called from `BlueprintFunctionHandler.cpp:160`
and `NetworkingHandler.cpp:510`.

A `parameters` array on a **built-in** eventType was dropped entirely by the graph path (the
authoring block sits inside `if (bIsCustomEvent)`) and still echoed in the response.

## The readback agreed with the write and both disagreed with the graph

The same request array was written into the blueprint registry (`:345`, `:358`, `:362`), and
`blueprint.get` merged registry events back into its response
(`Handlers/Blueprint/BlueprintInfoHandler.cpp:147-153`). So the verification step re-reported the
requested parameters — the closed loop where the instrument is wired to the same wrong thing as
the tool. The merge also appended any registry event whose name the graph snapshot does not
list, which is a positive claim that a node exists; `blueprint.add_event` followed by a
default-mode `blueprint.compile_bpir`, whose Phase 0 sweep deletes the entry
(`Docs/wiki-src/blueprint.bpir-gotchas.md:8`), produces exactly that state.

## History
- `#1-found-during-lying-verb-sweep` `OPEN` reporter — Source audit at `b92ba268`. Distinct from
  `E-add-event-no-node-id-echo` (Low, missing `nodeId`) and from
  `B-rpc-input-class-path-silent-wildcard`, whose fix explicitly did not touch `add_event`.
- `#2-validate-up-front-and-measure-pins` `IN-REVIEW` developer — **Source only; NOT compiled and
  NOT runtime-verified** (a map agent held the editor DLL; an integration pass owns the build).
  - `Handlers/Blueprint/BlueprintEventHandler.cpp`: `parameters` is parsed once up front with
    `ParseNamedTypePinParams` and gated **before any mutation** — non-object entries
    (`INVALID_ARGUMENT`, they were being skipped silently), an empty `name` (`INVALID_ARGUMENT`,
    named before typed so the message does not blame the type), a type that would become a wildcard
    pin (`TYPE_NOT_FOUND` via the shared `FindFirstPinParamWildcardFallback`), and `parameters` on a
    built-in eventType (`UNSUPPORTED_ARGUMENT`). The authoring loop now consumes the SAME parsed
    specs, so the thing validated is the thing written.
  - `FailedParams` is hoisted to handler scope and decides the response: a pin that still does not
    exist makes the call answer `PIN_CREATION_FAILED` with a `failedParameters[]` list, carried on
    the error's result object so the caller also sees which pins DO exist.
  - Both the response's `parameters` and the registry record are now built by `CollectEventPins`,
    the same graph reader `blueprint.get`'s snapshot uses — measured, never echoed.
  - `Handlers/Blueprint/BlueprintInfoHandler.cpp`: the events merge no longer unions registry-only
    events into `blueprint.get`. `BuildBlueprintSnapshot` always sets `events` from
    `CollectBlueprintEvents`, which walks every node of every `UbergraphPage`, so the snapshot is a
    complete enumeration and a registry event it does not list is an event the graph does not have.
    Only the degenerate no-snapshot fallback still reads the registry.
  - **No new error codes** — `TYPE_NOT_FOUND`, `INVALID_ARGUMENT`, `UNSUPPORTED_ARGUMENT` and
    `PIN_CREATION_FAILED` were all already in `Handlers/ErrorCodes.h`; the file now includes it and
    references the `ErrorCodes::ERR_*` constants for the new sites.
  - Tests: `Source/PinWright/Private/Tests/Blueprint/TestAddEventPinHonesty.cpp` — 5 automation
    tests. Four assert the failure direction (unresolvable type → `TYPE_NOT_FOUND` and no node;
    `parameters` on `BeginPlay` → `UNSUPPORTED_ARGUMENT` and no node added; empty param name →
    `INVALID_ARGUMENT` and no node; registry-only event absent from `blueprint.get`). One asserts
    the measured-not-echoed property with a deliberately padded pin name (`"  Good  "` in the
    request, `"Good"` in the response and on the node). All fail before the fix.
  - Docs: new `### blueprint.add_event` section in `Docs/wiki-src/blueprint.md` with the rejection
    table, and a paragraph under `### blueprint.get` stating `events` is a live graph enumeration.
  - **Known behaviour change to watch for:** `parameters` entries that previously produced a silent
    wildcard pin are now rejected. Any caller relying on that (e.g. passing `class:/Script/X.Y`) now
    gets `TYPE_NOT_FOUND` instead of a malformed pin — the same trade the `add_function` /
    `create_rpc_function` fix already made.
- `#3-compiled-and-suite-green` `IN-REVIEW` developer — Supersedes `#2`'s "NOT compiled" caveat.
  Built and tested in integration pass 8; committed as `eec42c96` and pushed. Clean module rebuild
  (all 7 module intermediates moved aside, `-DisableAdaptiveUnity -NoHotReloadFromIDE`):
  `Result: Succeeded`, zero errors and zero warnings in both the build log and UBT's `-Log=` target,
  no standalone `.cpp` actions, all 7 DLLs relinked. Both changed files present as named compile
  actions in their freshly generated unity blobs — `BlueprintEventHandler.cpp` in
  `Module.PinWright.9.cpp`, `BlueprintInfoHandler.cpp` in `Module.PinWright.10.cpp`.
  The link risk `#2` implied was checked and is not one: `CollectEventPins` is declared in the
  shared header `Handlers/Blueprint/BlueprintHandlerUtils.h:336` and defined non-static at
  `BlueprintHandlerUtils.cpp:1035` — it was never static or anonymous in `BlueprintInfoHandler.cpp`.
  Full suite **3723 tests performed, 3721 Success, 2 Fail** — the two pre-existing
  `localization.Validation.*` only; ZenServer probe 0; all five integration sub-modules loaded.
  All five new tests located by name in the log and `Result={Success}`: the four
  `PinWright.blueprint.add_event.*` plus `PinWright.blueprint.get.RegistryOnlyEventIsNotReported`.
  **Still not runtime-verified** through the MCP surface — no live `add_event` → `compile_bpir` →
  `blueprint.get` round trip was driven. Stays `IN-REVIEW`; a tester still has to close it, and the
  behaviour change flagged at the end of `#2` (previously-silent wildcard pins now `TYPE_NOT_FOUND`)
  is the thing to look for in the field.
