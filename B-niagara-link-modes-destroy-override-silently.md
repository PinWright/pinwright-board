---
id: B-niagara-link-modes-destroy-override-silently
title: "set_module_input's link and dynamicInput value modes destroy a pre-existing override with no opt-in and nothing in the response saying what they displaced"
status: DONE
severity: High
category: bug
tags: [niagara, set_module_input, dynamic-input, linked-parameter, override-pin, silent-mutation, no-opt-in]
encounters: 1
lastSeen: 2026-08-28T09:15:00+05:00
---

# The literal path now refuses to clobber a link; the other two paths still clobber silently

`B-niagara-literal-over-linked-override-pin` is fixed: writing a bare literal onto an override pin that
already has an inbound link is refused with `MODULE_INPUT_OVERRIDE_LINKED`, and replacing it is an
explicit `breakExistingLink: true` opt-in whose response carries
`replacedOverride: {valueMode, source}` naming what was displaced.

The `{link: ...}` and `{dynamicInput: ...}` value modes of the same verb still do the old thing: they
call `ClearModuleInputOverride` unconditionally, deleting whatever was on the pin -- a dynamic-input
chain an artist built, or another agent's parameter binding -- with no opt-in and nothing in the
response reporting it.

The inconsistency is the sharp edge. A caller who has learned that this verb refuses to clobber will
reasonably assume that holds for all three value modes. It does not.

**Fix:** the same `replacedOverride` report on both paths at minimum, so the mutation is disclosed.
Whether they should *also* gate behind `breakExistingLink` is a compatibility call: the literal path
could gate because writing a literal over a link was never a working operation, whereas assigning a
new dynamic input over an old one is plausibly intended most of the time.

## History
- `#1-family-inconsistency-after-the-literal-fix` `OPEN` reporter -- Raised by the agent that fixed
  `B-niagara-literal-over-linked-override-pin`, which scoped itself to the literal branch because the
  other two sit in code other agents were editing at the time. Source-level claim against the two
  `ClearModuleInputOverride` call sites.
- `#2-disclose-on-all-three-modes-ungated` `IN-REVIEW` developer -- Both call sites now route through a
  new `ClearOverrideAndRecordReplacement` helper that classifies the override pin with
  `NiagaraEdit::ClassifyModuleInputBindings` (the same walk feeding the `valueMode` readback) before
  clearing it, and the handler emits `replacedOverride {valueMode, source, value}` for all three value
  modes from one shared record (`FNiagaraLiteralOverrideReplacement` renamed to
  `FNiagaraOverrideReplacement`, gained a `Value` field for a displaced literal, which has no source to
  name). Deliberately NOT gated behind `breakExistingLink`: a link/dynamicInput write takes effect as
  asked, so gating would refuse the ordinary re-bind and the idempotent retry of a verb named "set" --
  argued in the branch comments and in the `value` param doc. Files:
  `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp`,
  `Source/PinWright/Private/Tests/Niagara/TestNiagaraSetModuleInputReplacedOverride.cpp` (new). Tests:
  `PinWright.niagara.set_module_input.LinkReportsReplacedOverride`,
  `PinWright.niagara.set_module_input.DynamicInputReportsReplacedOverride`.

- `#3-verified-against-built-binary` `DONE` verifier - Behavioural repro against the running editor (PinWright HEAD `b79ba53e`), 2026-08-28, UE 5.8, same scratch SimpleExplosion duplicate as `B-niagara-literal-over-linked-override-pin` `#3`. Both remaining value modes now disclose what they destroyed, ungated, in the live response. **dynamicInput over dynamicInput**: `set_module_input {emitter:"SimpleSpriteBurst", entryId:8B92C307...(ScaleColor), inputName:"Scale Alpha", value:{dynamicInput:"/Niagara/DynamicInputs/UniformRange/V2/RandomRangeFloat.RandomRangeFloat"}}` returned `success:true, dynamicInput:"...RandomRangeFloat"` plus `replacedOverride:{valueMode:"dynamicInput", source:"/Niagara/DynamicInputs/ValueFromCurve/FloatFromCurve.FloatFromCurve"}` - naming the chain it deleted. **link over literal**: `{entryId:FD5806F4...(AddVelocity), inputName:"Velocity Speed", value:{link:"User.VerifySpeed"}}`, on a pin then holding the literal `777.0`, returned `linked:true, parameter:"User.VerifySpeed", parameterType:"NiagaraFloat"` plus `replacedOverride:{valueMode:"local", value:"777.0"}` - so the `Value` field `#2` added to `FNiagaraOverrideReplacement` really does carry the displaced literal's pin default, which is the text a caller would restore. Neither write was gated behind `breakExistingLink`, which is what `#2` intended. Both `replacedOverride` classifications agree with the `moduleInputs[].valueMode` readback taken immediately before the write, so the write's account and a later `niagara.inspect` do not disagree.
