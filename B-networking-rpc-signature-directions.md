---
id: B-networking-rpc-signature-directions
title: "networking.create_rpc_function creates RPC parameter pins in the opposite graph directions"
status: IN-REVIEW
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

## Fix

The ticket was TRUE: `AddParsedPinParamsToNodes()` used the inverse of Unreal's
function-graph parameter directions and discarded every pin-creation result. The helper now
authors entry parameters with `EGPD_Output`, result parameters with `EGPD_Input`, reconstructs
the nodes, and returns a failure that makes `networking.create_rpc_function` remove the new
graph instead of reporting success.

`NetworkingHandler.cpp` now maps only `Server`, `Client`, and `NetMulticast` to exactly one
`FUNC_Net*` direction, applies `FUNC_NetReliable` only when requested, and refuses RPC return
values, unknown directions, and non-RPC reliability edits with `INVALID_RPC_CONFIGURATION`.
Blueprint validation enablement instead returns `UNSUPPORTED`. Unreal implements
`FUNC_NetValidate` through a
native thunk calling `_Validate`; a Blueprint graph cannot supply that contract, so the old flag-only
success response was unsafe. `Docs/wiki-src/networking.md` documents the refusal contract.
`TestNetworkingRpcSignature.cpp` adds the behavioural tests
`PinWright.networking.create_rpc_function.DirectionReliabilityMatrix` and
`PinWright.networking.create_rpc_function.InvalidConfigurationRefused` against transient
Blueprints.

Files changed: `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.h`,
`Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp`,
`Source/PinWright/Private/Handlers/Networking/NetworkingHandler.cpp`,
`Source/PinWright/Private/Handlers/ErrorCodes.h`,
`Source/PinWright/Private/Tests/Networking/TestNetworkingRpcSignature.cpp`, and
`Source/PinWright/Private/Tests/Networking/TestNetworkingHandlers.cpp`, and
`Docs/wiki-src/networking.md`. The runtime dispatch model and networking-info response shape were
deliberately left unchanged. No engine, packaged runtime, editor, build, or automation run was
performed; those remain for the independent compile/suite workers.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification compared `BlueprintHandlerUtils.cpp:1535-1560` with the working `BlueprintFunctionHandler.cpp:530-567` direction contract and confirmed the networking call at `NetworkingHandler.cpp:545-559`. No RPC reproduction, build, test, editor, or MCP call was run. Severity High because the normal authoring route can report success with a silently wrong signature.
- `#2-fixed-rpc-signature-flags` `IN-REVIEW` developer — Corrected function-entry/result pin directions with propagated failures, made RPC direction/reliability mapping explicit, and added typed pre-mutation refusals for invalid directions, return values, reliability targets, and unsupported Blueprint validation. Added transient-Blueprint behavioural coverage for all six direction/reliability combinations plus refusal atomicity. Source-reviewed only; compile and automation execution were deliberately left to the separate verification workers.
- `#3-addressed-read-only-review-risks` `IN-REVIEW` developer — Aligned validation discovery and wiki text with the `UNSUPPORTED` runtime refusal, restored direct `FUNC_NetValidate` readback coverage for legacy functions, and added injected post-graph-creation failure coverage for rollback. Source-reviewed only; no build, automation, editor, or MCP run was performed.
