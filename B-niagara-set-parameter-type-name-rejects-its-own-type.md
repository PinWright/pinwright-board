---
id: B-niagara-set-parameter-type-name-rejects-its-own-type
title: "niagara.set_parameter refuses the exact type name it reports: type 'NiagaraInt32' fails PARAMETER_TYPE_MISMATCH against 'NiagaraInt32', and first demands a JSON object for a scalar int; only the undocumented alias 'int32' works"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, set-parameter, type-names, error-message-contradicts-itself, undocumented-alias, discoverability]
encounters: 2
costly: 1
lastSeen: 2026-09-08T15:22:00Z
---

# The type name `niagara.inspect` prints is the one `set_parameter` rejects

`niagara.inspect` reports a parameter's type as `{"name": "NiagaraInt32", "struct":
"/Script/Niagara.NiagaraInt32"}`. Feeding that name straight back to `niagara.set_parameter` fails
twice, and the second failure contradicts itself.

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-07

`/Game/FPS/VFX/NS_Blood`, `Constants.Drips.SpawnBurst_Instantaneous.Spawn Count`
(existing, `NiagaraInt32`, value 1):

```
niagara.set_parameter {scope:"systemUpdateRapidIteration", name:"<above>",
                       type:"NiagaraInt32", value:90}
-> [INVALID_VALUE] Parameter type 'NiagaraInt32' requires a JSON object value.

niagara.set_parameter {... type:"NiagaraInt32", value:{"value":90}}
-> [PARAMETER_TYPE_MISMATCH] Parameter 'Constants.Drips.SpawnBurst_Instantaneous.Spawn Count'
   exists with type 'NiagaraInt32', not requested type 'NiagaraInt32'.

niagara.set_parameter {... type:"int32", value:90}
-> success
```

The second message names the same string on both sides of "not", so the comparison is not on the
name the message prints. The first pushes the caller toward an object wrapper that is not the
problem, and following it produces the self-contradicting error rather than the real one.

`niagara.add_parameter` accepts `type:"int32"` for the same type, so the alias is consistent between
the two verbs — it is simply the only spelling that works, and nothing documents it.

## Impact

Medium. No wrong data is written; the cost is diagnosis time and a plausible wrong turn. An agent
that reads the type off `niagara.inspect` — the documented discovery route — cannot write the
parameter back, and the error text actively misdirects: it blames the value shape, then denies an
identity.

## Fix directions

- Accept the canonical `Niagara*` names alongside the short aliases, or resolve both through one
  table so the accepted set matches what `inspect` prints.
- When the comparison fails, print the two things actually compared (resolved type definitions), not
  two identical display names.
- Raise `INVALID_VALUE` only after the type resolves, so a scalar int against an int parameter is
  never told it needs an object.
- Document the accepted `type` spellings on `niagara.set_parameter` and `niagara.add_parameter`.

## History

- `#2-type-object-is-a-silent-success-not-an-error` `OPEN` VFX builder — **severity Medium -> High.** The failures documented above are all loud. Handing `set_parameter` the type **object** that `inspect` actually returns is not: it reports `success: true` and writes nothing.

  The natural way to round-trip a store is to read it with `niagara.inspect` and feed each entry back. `inspect` returns `type` as an object:

  ```
  "type": {"name":"NiagaraInt32", "sizeBytes":4, "isDataInterface":false,
           "isUObject":false, "struct":"/Script/Niagara.NiagaraInt32"}
  ```

  Passing that object through as `type` gives, measured 2026-09-08 on `/Game/FPS/VFX/NS_Blood`, `/Game/FPS/VFX/NS_Explosion` and both `Scratch_FireballIso` rigs:

  ```
  niagara.set_parameter {scope:"systemUpdateRapidIteration",
                         name:"Constants.Drips.SpawnBurst_Instantaneous.Spawn Count",
                         type:{"name":"NiagaraInt32", ...}, value:90}
  -> success: true          <-- no error, no warning
  niagara.inspect ...
  -> value: 1               <-- unchanged
  ```

  Ten such writes across four systems all returned success and not one landed. The same calls with `type:"int32"` succeeded and stuck, confirming the only difference is the shape of `type`.

  This is worse than the `PARAMETER_TYPE_MISMATCH` and `INVALID_VALUE` paths in `#1`: those refuse and say so, so the caller retries. A `success: true` that writes nothing is only caught by an explicit read-back, and a caller who trusts the setter's own echo — which the `niagara.inspect` wiki page actively recommends ("prefer confirming from the setter's own echo ... over a second inspect") — will not catch it at all.

  Raising to High on impact class rather than reach: a setter that reports success without writing is a silent-data-loss shape, and the documentation currently steers callers away from the only check that detects it.

  Suggested fix: reject a non-string `type` with `INVALID_VALUE` naming the accepted aliases, or accept the object form by reading its `name`/`struct` — either is fine, but `success: true` with no write is not.

  Cost: ten no-op writes inside a world-lock slot, found only because the restore script re-read the store afterwards; without that read-back the follow-on capture pass would have photographed template defaults and been reported as a content result.

- `#3-one-resolver-for-both-verbs` `IN-REVIEW` developer — **#1 fixed; #2 could not be reproduced from source and is now pinned instead.**

  Root cause of #1 was two tables answering "what type is this?" for one request. `NiagaraEditTypes.cpp` decided the VALUE SHAPE from the raw string and fell through to "is it a registered script struct?" — true of `NiagaraInt32` and `NiagaraFloat` as much as of `NiagaraSpawnInfo`, which is why a scalar int was told it needed a JSON object. `NiagaraEditHandler.cpp` then resolved the same string through a registry walk matching candidates by their STORAGE struct; `FNiagaraTypeDefinition::GetStruct()` of an **enum** type returns `FNiagaraInt32`, so every registered enum answers to `NiagaraInt32` and to `/Script/Niagara.NiagaraInt32` and the winner is registration order (i.e. which content enums the editor loaded). When an enum won, the comparison failed — and the message printed the existing type's display name against the caller's raw request string, so both sides read the same text no matter which side was wrong.

  New `Source/PinWright/Private/Handlers/Niagara/NiagaraParameterTypeResolver.{h,cpp}` (`PinWrightNiagara::ResolveNiagaraParameterType`) is the single resolver: an alias table carrying every canonical name `inspect` prints (`NiagaraFloat`, `NiagaraInt32`, `NiagaraBool`, `Vector2f`, `Vector3f`, `NiagaraPosition`, `LinearColor`, `NiagaraID`, `NiagaraSpawnInfo`, `NiagaraHalf`, `NiagaraHalfVector2/3/4`) alongside the pre-existing short aliases, then a registry walk that prefers a candidate's own identity name over its storage struct and never lets an enum answer for a struct name or path. It returns a value KIND, so `ValidateTypedValue` (`NiagaraEditTypes.cpp`) now resolves first and derives the shape from the resolved type — scalars take scalars — and `SetTypedParameterValue` / `ApplyParameterMutation` (`NiagaraEditHandler.cpp`) dispatch on that same resolution instead of re-parsing the string. The removed duplicates were `TryParseNiagaraTypeName`, `ResolveRegisteredNiagaraScriptStructType`, `ResolveNiagaraParameterType` (handler) and `IsRegisteredNiagaraScriptStructType` (types). Side effects of the merge: `half`, `vec2`, `SpawnInfo`, `NiagaraID` and the half-vector spellings, all supported by the writer but refused by the old validator, now validate. `PARAMETER_TYPE_MISMATCH` prints both sides as `name (path)` plus the raw request spelling. The `type` param description on both verbs and the `niagara` wiki-src page (`docs/wiki-src/niagara.md`, `### niagara.set_parameter` plus a new `### niagara.add_parameter`) now carry the whole table.

  **#2 (`type` OBJECT → `success: true`, nothing written) does not exist on this tree, and the history entry is the record of why rather than a fix.** Traced both layers of the exact payload. Layer 1: `type` is declared `string` on both verbs, and `FRpcDispatcher::ValidateHandlerParams`' declared-type gate (`Dispatch/RpcDispatcher.cpp`, `Handlers/ParamTypeCheck.h`, landed 2026-09-05 for `B-param-type-never-validated`) refuses an object with `PARAM_TYPE_MISMATCH` naming `'type'` before the handler body runs — reused, not duplicated. Layer 2: `ParseParameterPayload` reads `type` with `TryGetStringField`, which answers `""` for an object, and the empty-type branch refuses `INVALID_ARGUMENT`. So the measured `success: true` cannot come from either path as they stand; the 2026-09-08 editor was running a pre-gate build, and even that build refused with `Missing parameter 'type'.` — which is the one thing that WAS wrong and is fixed here: a present-but-mis-shaped `type` no longer reports as absent, it names the field and says to send `inspect`'s `type.name` rather than the object.

  Regression tests, `Source/PinWright/Private/Tests/Niagara/TestNiagaraParameterTypeNames.cpp`:
  - `PinWright.niagara.parameter_types.InspectNamesResolveToTheirEngineTypes` — every canonical name resolves to its engine definition with the right value kind. Counterfactual: drop the alias rows and `NiagaraHalfVector2/3/4` stop resolving at all (`FNiagaraTypeRegistry` never registers them); drop the enum exclusion and `/Script/Niagara.NiagaraInt32` can resolve to an enum type.
  - `PinWright.niagara.set_parameter.CanonicalTypeNameWritesScalar` — `type: "NiagaraInt32", value: 90` writes 90, asserted on the store. Counterfactual: revert `ValidateTypedValue` to the string-keyed order and the call is refused `INVALID_VALUE` ("requires a JSON object value") before any write.
  - `PinWright.niagara.set_parameter.TypeMismatchNamesResolvedTypes` — the refusal names `/Script/Niagara.NiagaraFloat` and `/Script/Niagara.NiagaraInt32`. Counterfactual: revert the message and it prints the display name against the raw string `int32`, so neither path appears.
  - `PinWright.niagara.set_parameter.ObjectTypeIsRefusedNotSilentlyAccepted` — the inspect type object is refused at the dispatcher (`PARAM_TYPE_MISMATCH`, names `'type'`, store unchanged) and at the parser (`INVALID_ARGUMENT`, names `'type'` and points at `'name'`). Counterfactual: revert the parser message and the `'name'` assertion fails; widen the declared type to accept objects and the dispatcher assertion fails.
