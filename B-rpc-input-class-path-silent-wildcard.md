---
id: B-rpc-input-class-path-silent-wildcard
title: "create_rpc_function / add_function silently turn the documented `class:/Script/X.Y` input type into a wildcard pin (success, then uncompilable BP)"
status: IN-REVIEW
severity: High
category: bug
tags: [networking, create-rpc-function, add-function, pin-type, wildcard, silent-fail, class-path]
---

# `create_rpc_function` / `add_function` inputs silently degrade the documented `class:/Script/X.Y` token to a wildcard pin

`networking.create_rpc_function` (and `blueprint.add_function`, whose
`inputs[].type` grammar it reuses) **accepts** the input pin type token
`class:/Script/Engine.Actor` — the exact form its own docs point you to — returns
`success:true`, echoes the type back verbatim, but materializes the pin as an
**undetermined `wildcard`**, not the documented object/class reference. The
blueprint then fails to compile with a "type is undetermined" error that surfaces
on a *later* `blueprint.compile` call, far from the `create_rpc_function` call
that actually introduced the bad pin.

This is a silent success-with-wrong-effect: a documented, valid-looking input
produces a malformed result with no error at the call site.

## Why the token looks valid (documented contract)

- `networking.create_rpc_function` `inputs` doc: *"type accepts the same tokens
  as blueprint.add_function's inputs."*
- `blueprint.add_function` `inputs[].type` doc: *"type accepts the same tokens as
  blueprint.add_variable's variableType."*
- `blueprint.add_variable` `variableType` doc: *"...or `'class:/Script/X.Y'` for
  object/class refs..."*

So the documentation chain explicitly tells a caller to use
`class:/Script/Engine.Actor` for an Actor object/class reference input pin.

## The asymmetry that makes this a bug, not just a doc gap

The **same** `class:/Script/Engine.Actor` token is handled three different ways by
three handlers that all claim the same type vocabulary:

| handler | result for `class:/Script/Engine.Actor` |
|---|---|
| `blueprint.add_variable` (`variableType`) | clean `[TYPE_NOT_FOUND]` rejection (loud, correct) |
| `networking.create_rpc_function` (`inputs[].type`) | `success:true`, pin becomes `wildcard` (silent, wrong) |
| `blueprint.add_function` (`inputs[].type`) | same silent `wildcard` (shared grammar) |

`add_variable` already refuses the token (see `E-add-variable-type-format` — that
ticket is about `add_variable`'s *clean rejection* of `/`-leading / `class:` path
forms). The RPC/function input path instead falls through to a wildcard pin and
reports success. The correct, non-wildcard form is `object<Actor>`
(`bpir.types.md`: `object<ClassName>` for a UObject reference) — that compiles
clean.

## Verbatim repro (replayed on `mcp__editor-automation__call`)

Fresh BP `/Game/ReplayTest/BP_ReplayPickup` (parent `Actor`):

1. Create RPC with the documented `class:` token — reported success:
   ```
   networking.create_rpc_function {
     "blueprintPath": "/Game/ReplayTest/BP_ReplayPickup",
     "functionName": "ServerTestClassColon", "rpcType": "Server", "reliable": true,
     "inputs": [ { "name": "ClassRef", "type": "class:/Script/Engine.Actor" } ] }
   → { "success": true, "functionName": "ServerTestClassColon", ...,
       "inputs":[{"name":"ClassRef","type":"class:/Script/Engine.Actor"}], ... }
   ```
   (no warning field; the type is echoed back as if it resolved)

2. Control with the BPIR grammar form — also reported success:
   ```
   networking.create_rpc_function { ... "functionName":"ServerTestObjectForm",
     "inputs":[{"name":"Instigator","type":"object<Actor>"}] } → success:true
   ```

3. `blueprint.compile { "blueprintPath":"/Game/ReplayTest/BP_ReplayPickup" }`
   → `compiled:false, status:"Error",` exactly ONE error, on the `class:` pin only:
   ```
   "The type of  Class Ref  is undetermined.  Connect something to
    ServerTestClassColon  to imply a specific type."
   ```
   (the `object<Actor>` function compiles clean — no error for `Instigator`)

4. Decompile confirms the actual pin types produced:
   ```
   blueprint.decompile_function ServerTestClassColon
     → "entry function ServerTestClassColon(wildcard ClassRef) ..."
   blueprint.decompile_function ServerTestObjectForm
     → "entry function ServerTestObjectForm(object<Actor> Instigator) ..."
   ```
   `class:/Script/Engine.Actor` → `wildcard` (broken); `object<Actor>` → correct.

5. Contrast — `add_variable` rejects the same token cleanly:
   ```
   blueprint.add_variable { ..., "variableType":"class:/Script/Engine.Actor" }
   → [TYPE_NOT_FOUND] Could not resolve variableType 'class:/Script/Engine.Actor'.
      Accepted forms: ... wrappers (... object<T> ... class<T> ...)
   ```

## Impact

An agent following the documented type vocabulary builds an RPC whose Actor input
is silently a wildcard, gets a clean `success`, and only discovers the breakage at
a later compile with a confusing "undetermined type / connect something" message
that never names the offending `create_rpc_function` call or the bad token. The
recovery is remove-and-recreate the RPC with `object<Actor>` (observed in the
field as exactly this remove_function → recreate cycle).

## What it should do

The `inputs[].type` / `outputs[].type` parser on `create_rpc_function` and
`add_function` should treat an unresolved type token the same way `add_variable`
does: **reject it loudly** with `TYPE_NOT_FOUND` (listing accepted forms) instead
of silently substituting a wildcard pin and returning success. Ideally also accept
the documented `class:/Script/X.Y` path form by routing it through the path-aware
`ResolveUClass`/`ResolveUScriptStruct` (the same fix `E-add-variable-type-format`
#6 proposes for `add_variable`), so the documented token resolves to a real
object/class pin. At minimum the response must carry a `warning` (or hard error)
whenever an input pin falls back to wildcard, so the failure is visible at the call
that caused it rather than at a downstream compile.

**Workaround:** use the BPIR grammar form `object<Actor>` (UObject ref) or
`class<Actor>` (class ref) for object/class input pins — not the documented
`class:/Script/Engine.Actor` colon-path form.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed on a fresh BP
  `/Game/ReplayTest/BP_ReplayPickup` via `mcp__editor-automation__call`.
  `networking.create_rpc_function` with `inputs:[{name:ClassRef,
  type:"class:/Script/Engine.Actor"}]` returned `success:true` (type echoed back,
  no warning), but `blueprint.decompile_function` shows the entry pin is
  `wildcard ClassRef`, and `blueprint.compile` fails with "The type of Class Ref is
  undetermined. Connect something to ServerTestClassColon to imply a specific
  type." A sibling RPC built the same way with `object<Actor>` decompiles to
  `object<Actor> Instigator` and compiles clean. The same `class:/Script/Engine.Actor`
  token is *cleanly rejected* by `blueprint.add_variable` (`TYPE_NOT_FOUND`), so the
  three handlers sharing one documented type vocabulary disagree: add_variable
  rejects loudly, while create_rpc_function/add_function inputs silently degrade to
  a wildcard pin and report success. Distinct from `E-add-variable-type-format`
  (that ticket = add_variable's clean rejection of path forms) and from
  `E-create-rpc-function-no-param-slot` (that ticket = create_rpc_function lacking
  an inputs slot at all, since fixed); this is the *inputs-slot-now-exists-but-
  silently-wildcards* bug. Proposed: reject unresolved input/output type tokens
  loudly (or surface a wildcard-fallback warning), and/or accept the documented
  `class:/Script/X.Y` path via the path-aware resolvers.
- `#2-implement` `IN-REVIEW` developer — Implemented the minimum (loud-rejection)
  half: the input/output pin parser on `networking.create_rpc_function` and the
  `blueprint.add_function` new-function path now reject an unresolved type token with
  `[TYPE_NOT_FOUND]` (listing accepted forms, matching `add_variable`) instead of
  silently building a wildcard pin and returning `success:true`. Added a shared
  helper `BlueprintHandlerUtils::FindFirstPinParamWildcardFallback` that scans the
  already-parsed `FParsedPinParam` list and flags the first token that would degrade
  to wildcard — both a parse miss (`bParseOk==false`, e.g. the documented-but-
  unsupported `class:/Script/X.Y` colon-path form) and a token that parses but
  resolves to no UClass/UEnum/UScriptStruct (via `BuildNamedPinDescriptor`, the same
  PC_Wildcard check `add_variable` makes). The handlers call it after parsing and
  before any graph work, sending `TYPE_NOT_FOUND` at the call site. The `add_function`
  override path is left to its existing stricter signature-match rejection (guarded
  `if (!bOverride)`). The SECONDARY ask — actually *accepting* the `class:/Script/X.Y`
  path form via path-aware resolvers — is deliberately deferred to
  `E-add-variable-type-format` #6 (shared BPIR-tokenizer change; the inputs path does
  not route through `MakePinTypeFromBpirText`, so that fix would not auto-cover it
  either, but the grammar change belongs there).
  Files: `Private/Handlers/Blueprint/BlueprintHandlerUtils.h` (+helper decl),
  `Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp` (+helper impl),
  `Private/Handlers/Networking/NetworkingHandler.cpp` (create_rpc_function reject),
  `Private/Handlers/Blueprint/BlueprintFunctionHandler.cpp` (add_function new-function
  reject). Test: `Private/Tests/Core/TestMakePinType.cpp` →
  `FPinParamRejectsClassColonPathTest`
  (`EditorAutomationRpcGateway.core.make_pin_type.PinParamRejectsClassColonPath`)
  drives the production helper through the production `ParseNamedTypePinParams`
  AllowWildcardFallback path: asserts `class:/Script/Engine.Actor` and a bare unknown
  identifier are rejected (with the token named + accepted forms in the message),
  while `object<Actor>` and an empty list are accepted — fails if the helper or its
  wiring is reverted.
