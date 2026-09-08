---
id: B-niagara-set-parameter-type-name-rejects-its-own-type
title: "niagara.set_parameter refuses the exact type name it reports: type 'NiagaraInt32' fails PARAMETER_TYPE_MISMATCH against 'NiagaraInt32', and first demands a JSON object for a scalar int; only the undocumented alias 'int32' works"
status: OPEN
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
