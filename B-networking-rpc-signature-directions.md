---
id: B-networking-rpc-signature-directions
title: "networking.create_rpc_function creates RPC parameter pins in the opposite graph directions"
status: OPEN
severity: High
category: bug
tags: [networking, rpc, blueprint, function-signature, pin-direction, false-success]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# RPC creation reverses Blueprint parameter graph directions

## What happens

The shared `AddParsedPinParamsToNodes()` helper adds request inputs to the function
entry with `EGPD_Input` and outputs to the result with `EGPD_Output`
(`Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp:1535-1560`).
The working `blueprint.add_function` path does the opposite: entry parameters are
output pins and result parameters are input pins
(`BlueprintFunctionHandler.cpp:530-567`).
`networking.create_rpc_function` calls the faulty shared helper after setting the
network flags (`Handlers/Networking/NetworkingHandler.cpp:545-559`) and continues
to compile and report success.

## Why it matters

A normal RPC-authoring call can return success while creating a signature whose
parameters face the wrong way. Callers then build logic against a result they trust
but cannot use correctly. Severity is High under the silent-wrong-data rule.

## What should happen

Use `EGPD_Output` for function-entry inputs and `EGPD_Input` for function-result
outputs, propagate pin-creation failures instead of continuing, reconstruct the
nodes as required, and test both directions on a real RPC function.

## Workaround

Author the function through the working lower-level Blueprint function path and set
network flags separately, or repair the signature manually in the editor.

## Related

- `B-add-function-outputs-become-inputs`
- `B-interface-function-with-outputs-unimplementable`
- `B-bpir-single-named-output-forced-returnvalue`

These are the wave-6 function-output tickets whose grouped review exposed the shared
RPC helper defect.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification compared `BlueprintHandlerUtils.cpp:1535-1560` with the working `BlueprintFunctionHandler.cpp:530-567` direction contract and confirmed the networking call at `NetworkingHandler.cpp:545-559`. No RPC reproduction, build, test, editor, or MCP call was run. Severity High because the normal authoring route can report success with a silently wrong signature.
