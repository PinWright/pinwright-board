---
id: B-niagara-module-input-cannot-link-particle-attribute
title: "niagara.set_module_input cannot bind a module input to a built-in attribute (Particles.NormalizedAge): it infers the link type from a parameter store instead of from the input"
status: OPEN
severity: High
category: bug
tags: [niagara, set-module-input, link, particle-attribute, normalized-age, over-life, hard-blocker]
encounters: 1
lastSeen: 2026-09-02T19:40:00Z
---

# The single most common VFX authoring move — drive an input from normalized age — is unreachable

`niagara.set_module_input` documents `value: { link: "User.X" }` for binding a parameter. Binding one
of Niagara's built-in particle attributes is refused:

```
niagara.set_module_input {assetPath: "/Game/FPS/VFX/Emitters/E_FPS_MuzzleAR_Smoke",
                          entryId: <ScaleSpriteSize>, inputName: "Uniform Scale Factor",
                          value: {link: "Particles.NormalizedAge"}}
  -> [PARAMETER_NOT_FOUND] Cannot infer a type for linked parameter 'Particles.NormalizedAge'
     (module input type unknown).
```

The refusal is not about the attribute. It is about type inference: the handler looks the link name up
in a parameter store to learn its type, and `Particles.*` / `Engine.*` attributes are not in any store
on a standalone emitter asset (nor before a compile on a system). The error text says as much —
"module input type unknown" — even though the input's type is published by the same plugin, in
`niagara.inspect {includeStack:true}` → `stack.modules[].moduleInputs[].type`
(`"NiagaraFloat"` here). The information needed to type the link is already available and simply is
not consulted. Reproduced on a clean input (override reset first), so it is not a
side effect of an existing dynamic-input override.

Impact: "scale this over the particle's life" — size growth, colour ramps, width falloff, drag ramps —
is how essentially every real effect is authored, and the standard way to express it through stock
modules is `<input> ← Particles.NormalizedAge` (directly, or as the `Alpha` of a `Lerp_*` dynamic
input). None of that is reachable. The remaining routes are all worse:

- `Lerp_Float.Alpha` via the dotted sub-input handle fails with the **same** error, so nesting does not
  escape it.
- The curve modes (`ScaleSpriteSize` "Uniform Curve") depend on a stock curve whose shape cannot be
  read or edited: `niagara.set_curve_keys {scope:"updateRapidIteration", parameterName:
  "ScaleSpriteSize.Uniform Curve Sprite Scale"}` returns `DATA_INTERFACE_NOT_FOUND` (module-input curve
  DIs are not in a rapid-iteration store), `asset.dump` of the emitter emits no curve keys, and
  `niagara.inspect` on the module script asset returns three fields and nothing else. So the authored
  behaviour is unverifiable — the module either ramps or is an identity no-op and the caller cannot
  tell which without spawning the system.

That combination is what makes this High rather than Medium: there is no route to a *verified*
over-life curve on a module input through this namespace at all.

**Workaround:** none that produces a verified result. Authoring the effect by hand in the Niagara
editor is the only reliable path today.

**Fix:** type the link from the module input's own declared type (the value `moduleInputs[].type`
already reports) and fall back to the store lookup only when the input's type is genuinely
indeterminate. Accepting an explicit `linkType` on the call would also unblock it.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8, shared editor, port 27145) building
  muzzle-smoke emitters under `/Game/FPS/VFX/Emitters/`. Evidence: the refusal above on
  `ScaleSpriteSize.Uniform Scale Factor` (type `NiagaraFloat` per the same session's
  `niagara.inspect`), the identical refusal on
  `ScaleSpriteSize.Uniform Scale Factor.Lerp_Float.Alpha`, and the `DATA_INTERFACE_NOT_FOUND` from
  `niagara.set_curve_keys` recorded above. No plugin source read; diagnosis is from the RPC responses.
