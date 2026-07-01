---
id: E-create-rpc-function-no-param-slot
title: "networking.create_rpc_function has no inputs/outputs slot — the RPC's parameters (e.g. a damage amount) can't be declared via the typed surface"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [networking, create-rpc-function, rpc, parameters, signature, add-function-parity]
---

# `networking.create_rpc_function` can't declare the RPC's parameters — a Server RPC that takes an argument is half-buildable through the typed surface

`networking.create_rpc_function` accepts only `blueprintPath`, `functionName`,
`rpcType`, and `reliable` (NetworkingHandler.cpp:380-385). It creates the
function graph and stamps the net flags
(`FUNC_Net` + `FUNC_NetServer`/`FUNC_NetClient`/`FUNC_NetMulticast`, plus
`FUNC_NetReliable`) on the `UK2Node_FunctionEntry` via `AddExtraFlags`
(NetworkingHandler.cpp:410-415), then compiles. But it never touches the
function's **parameter signature** — the new RPC is created with an empty
parameter list and there is no slot to add one.

Real RPCs almost always carry arguments. The canonical example is exactly the
one this task hit: a `ServerApplyDamage(float DamageAmount)` reliable Server
RPC. Through the typed networking surface you can create the function and make
it a reliable validated Server RPC, but you **cannot give it the damage-amount
input** — so the RPC that gets built is `ServerApplyDamage()`, which is not the
RPC the task asked for. The agent is forced to leave the signature empty or
drop to a different surface.

## Why the sibling surface doesn't close the gap (a split-capability seam)

`blueprint.add_function` already does the parameter half:
> "Create a new UFunction graph on a Blueprint with caller-specified
> input/output pins" — with `inputs`/`outputs` arrays of `{name, type}` pin
> definitions (BlueprintFunctionHandler.cpp:65-71).

…but it has **no `rpcType`/`reliable` net-flag support**, so a function created
there is a plain BlueprintCallable function, not an RPC. The two halves of the
single intent "an RPC function that takes arguments" live on two different
methods, and **neither method can produce the result alone**:

| method | declares parameters? | sets RPC net flags? |
|---|---|---|
| `networking.create_rpc_function` | no | yes |
| `blueprint.add_function` | yes (`inputs`/`outputs`) | no |

There is no documented typed path that combines them. (`set_rpc_reliability` /
`configure_rpc_validation` are post-hoc togglers that operate on a function by
name; they don't add parameters, and re-running `blueprint.set_function_settings`
covers access/pure/const/category, not the Net flags either.) The only way to
get a parameterized RPC through the MCP is a BPIR/graph workaround that
hand-authors the entry node's pins — well outside the obvious typed verb.

## What it should do

Add optional `inputs` (and, for symmetry, `outputs`) pin-definition arrays to
`networking.create_rpc_function`, accepting the **same `{name, type}` token
shape `blueprint.add_function` already parses** (so the type vocabulary is one
source of truth). Then a single typed call builds the whole RPC:

```jsonc
networking.create_rpc_function {
  "blueprintPath": "/Game/Global/Blueprints/PlayerCharacter",
  "functionName": "ServerApplyDamage",
  "rpcType": "Server",
  "reliable": true,
  "inputs": [ { "name": "DamageAmount", "type": "float" } ]
}
```

Implementation hint: the function graph is already created here via
`FBlueprintEditorUtils::CreateNewGraph` + `AddFunctionGraph` and the entry node
is already located in the `for (UEdGraphNode* Node : NewGraph->Nodes)` loop
(NetworkingHandler.cpp:399-418). The same `UK2Node_FunctionEntry` that receives
`AddExtraFlags(NetFlags)` is where `blueprint.add_function` adds user-defined
pins — reuse that handler's input-pin construction path against this entry node
before the compile, so parameter parsing stays identical across the two verbs.

**Acceptance check:** after the call above, `blueprint.inspect` /
`blueprint.decompile_function` on `ServerApplyDamage` shows a `DamageAmount`
float input pin on the entry node, and the function still carries the Net /
NetReliable / NetServer flags.

**Workaround until implemented:** create the RPC with
`networking.create_rpc_function` (for the net flags), then author the parameter
via a BPIR/graph edit to the function-entry node — two surfaces and a
non-obvious hand-edit for what reads as one typed intent.

## Evidence
Task `networking.get_networking_info` (focus), a clean ~33-call end-to-end
build of `/Game/Global/Blueprints/PlayerCharacter` (baseline read → add 2
replicated floats → set RepNotify + replication condition → create reliable
validated Server RPC → bump net update frequency → readback) with **zero
is_error, zero retries, zero wiki-nav loops, no python fallback** — the mutation
path was smooth. The friction note's second distinct point (the first, the
readback gap, is `F-networking-info-no-rpc-detail`): *"create_rpc_function has
no slot for the damage parameter, so the RPC's damage-amount input could not be
added via the typed surface."* The task's story step 5 explicitly asked for
`ServerApplyDamage` to *"take the damage amount"*; the typed surface built a
parameterless `ServerApplyDamage()` instead, so the requested signature was
unreachable without leaving the networking namespace.

## History
- `#1-initial-audit` `OPEN` reporter — Filed as the ergonomic/process sibling of the readback ticket `F-networking-info-no-rpc-detail` (which the per-finding judge already filed for this same task). Distinct angle: not "can't read it back" but "can't author it in the first place." `networking.create_rpc_function` (NetworkingHandler.cpp:380-385) takes only blueprintPath/functionName/rpcType/reliable and creates an empty-signature RPC — no `inputs`/`outputs` slot — while `blueprint.add_function` (BlueprintFunctionHandler.cpp:65-71) has `inputs`/`outputs` but no net-flag support, so the single intent "RPC that takes arguments" is split across two verbs and neither can produce it alone. Task asked for `ServerApplyDamage(damage)`; typed surface could only build `ServerApplyDamage()`. Proposed adding `inputs`/`outputs` (reusing `blueprint.add_function`'s pin-parsing path against the same `UK2Node_FunctionEntry` that already receives the net flags at NetworkingHandler.cpp:399-418).
- `#2-implement` `IN-REVIEW` developer — Implemented the root-cause fix. Added optional `inputs`/`outputs` `{name,type}` pin-definition arrays to `networking.create_rpc_function`'s param spec and handler body (`NetworkingHandler.cpp`). The body now parses them with the shared `BlueprintHandlerUtils::ParseNamedTypePinParams` (same type vocabulary as `blueprint.add_function`), then materializes pins via `BlueprintHandlerUtils::AddUserDefinedPin` against the located `UK2Node_FunctionEntry` (inputs) / `UK2Node_FunctionResult` (outputs, falling back to the entry node) — added right after `AddExtraFlags(NetFlags)` and before the existing `CompileBlueprint`, so net flags and parameters coexist. Pulled in `K2Node_FunctionResult.h` + `Handlers/Blueprint/BlueprintHandlerUtils.h`; parse misses fall back to a wildcard pin with a warning, mirroring `add_function`. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Networking/NetworkingHandler.cpp`. Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Networking/TestNetworkingHandlers.cpp` — new `EditorAutomationRpcGateway.networking.create_rpc_function.InputParamPinCreated` creates a real package-backed blueprint, calls the handler with a reliable Server RPC `ServerApplyDamage` + `inputs=[{DamageAmount,float}]`, and asserts the entry node carries BOTH the `DamageAmount` float input pin AND the surviving `FUNC_Net`/`FUNC_NetReliable`/`FUNC_NetServer` flags (fails if the inputs handling is reverted). Note for a follow-up: the near-identical `misc.create_rpc` (MiscHandler.cpp) has the same empty-signature gap and was left untouched here.
