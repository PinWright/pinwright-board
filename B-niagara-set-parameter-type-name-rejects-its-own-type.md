---
id: B-niagara-set-parameter-type-name-rejects-its-own-type
title: "niagara.set_parameter refuses the exact type name it reports: type 'NiagaraInt32' fails PARAMETER_TYPE_MISMATCH against 'NiagaraInt32', and first demands a JSON object for a scalar int; only the undocumented alias 'int32' works"
status: OPEN
severity: Medium
category: bug
tags: [niagara, set-parameter, type-names, error-message-contradicts-itself, undocumented-alias, discoverability]
encounters: 1
lastSeen: 2026-09-07T07:15:00Z
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
