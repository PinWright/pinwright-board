---
id: B-effect-niagara-activation-not-measured
title: "effect.activate_niagara and deactivate_niagara report a hardcoded active flag, never a measured one"
status: OPEN
severity: High
category: bug
tags: [niagara, effect, false-success, measured-vs-requested]
---

# effect.activate_niagara and deactivate_niagara report a hardcoded active flag, never a measured one

`effect.activate_niagara` writes `Resp->SetBoolField(TEXT("active"), true)` immediately after
calling `Activate(bReset)`, and `effect.deactivate_niagara` writes a hardcoded `false` the same
way (`Handlers/VFX/EffectHandler.cpp`, the two `SetBoolField(TEXT("active"), ...)` sites). Neither
reads `UNiagaraComponent::IsActive()` back. The field is an echo of what the verb was asked to do,
not a statement about what happened.

That makes `active` unfalsifiable: it reads `true` whether the component activated, silently
no-oped, or was already dead. `Activate()` returns void and has several paths that leave a
component inactive — no asset, an invalid or uncompiled system, a component that is not
registered, or a system whose emitters are all disabled. All of them report `active: true`.

This is the same failure class as `B-lighting-configure-shadows-echoes-unmeasured` and
`B-lighting-writes-vetoed-by-scalability-cvars`, both fixed on 2026-08-28 by adopting the
measured-vs-requested pattern: publish what was read back, name the requested value separately,
omit rather than zero when the measurement cannot be made, and warn when the request did not take
effect.

It also defeats the one oracle a caller currently has. `object.call_function IsActive` is the only
way to learn the truth today, which means the verb's own response cannot be trusted for the exact
thing the verb exists to do.

**Found by:** the agent implementing `B-niagara-validate-green-while-component-inactive`, which
lists this as its defect 3. That ticket's other three defects remain open there; this one is split
out because it is a silent false-success in its own right rather than a validate-reporting gap,
and because it is fixable independently.

**Workaround:** ignore the `active` field and follow every call with
`object.call_function` `IsActive` on the component, or with `niagara.validate`, which as of
2026-08-28 publishes a measured `componentActivation` block.

**Fix:** read `IsActive()` back after the call and publish the measured value as `active`, with the
requested state named separately. Warn when they disagree — a request that did not take effect is
the case the caller most needs to hear about. Where no component could be resolved, omit `active`
rather than reporting `false`, so "not measured" and "measured inactive" stay distinguishable.
`Handlers/Environment/LightingHandler.cpp` carries the shipped shape of this pattern.

## History
- `#1-hardcoded-active-flag` `OPEN` reporter — "effect.activate_niagara sets active:true immediately after Activate(bReset) with no readback, and deactivate_niagara hardcodes false the same way; both verified in EffectHandler.cpp at the two SetBoolField(TEXT(\"active\")) sites. Split out of B-niagara-validate-green-while-component-inactive defect 3, which stays open for its remaining defects."
