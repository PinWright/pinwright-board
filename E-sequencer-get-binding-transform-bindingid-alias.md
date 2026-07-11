---
id: E-sequencer-get-binding-transform-bindingid-alias
title: "sequencer.get_binding_transform requires 'bindingId' and rejects the 'binding'/'bindingGuid' alias its sibling sequencer verbs accept"
status: OPEN
severity: Low
category: ergonomic
tags: [sequencer, param-alias, get_binding_transform, bindingId, binding, drift]
encounters: 1
lastSeen: 2026-07-11T08:14:20+03:00
---

# sequencer.get_binding_transform rejects the `binding` alias its siblings accept

Binding-parameter naming is inconsistent inside the `sequencer` namespace. The
control-rig / binding verbs the caller had just read and used —
`sequencer.add_controlrig_track`, `sequencer.list_controls`,
`sequencer.key_controls`, `sequencer.get_control_value` — accept the binding
GUID under a `binding` key (the call log shows `add_controlrig_track {binding:...}`
parsing the GUID and failing downstream on `[BINDING_NOT_SKELETAL]`, i.e. the
`binding` key was honored). But `sequencer.get_binding_transform` requires the
GUID under exactly `bindingId` and rejects `binding` with a hard
`[MISSING_REQUIRED_PARAM]`. A caller who carries the same GUID variable across
sibling calls eats a wasted round-trip on the one verb whose contract drifted.

**Repro:** `sequencer.get_binding_transform {binding:"EB7E779D449399A38C3A8DA6CA0FC15B", frame:0}`
-> `[MISSING_REQUIRED_PARAM] Missing required parameter 'bindingId' (type: string)`.
Retry with `{bindingId:"EB7E..."}` is the accepted shape (`get_binding_transform`
was implemented with the `bindingId` canonical per `F-sequencer-evaluate-readback`
`#2`). The canonical name is fine; the missing `binding`/`bindingGuid` alias is
the friction.

Same param-name-drift class as the established `param-alias` family
(`E-blueprint-param-name-path-vs-assetpath` DONE, `E-material-editor-param-name-drift`
DONE, `E-actor-verbs-reject-actorpath-slot`, `E-level-create-name-path-alias`),
but here the drift is a *cross-method inconsistency within one namespace*: the
control-rig verbs and the transform-readback verb disagree on the binding-GUID
key name, so no single doc read protects the caller — using one verb's contract
against the sibling is what breaks.

**Fix direction:** standardize the accepted binding-GUID alias set across the
sequencer namespace so `get_binding_transform` honors `binding` / `bindingGuid`
in addition to its canonical `bindingId` (mirror the `ParamAliasUtils`
alias-spec machinery used by the `param-alias` sweep). Keep the canonical name;
add the aliases so callers can pass the same key to every sequencer binding verb.

severity rationale: impact=discoverability/naming (self-recoverable single param
slip, no wrong data) x reach=rare (transform-readback verify verb, not
every-session) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `sequencer.key_controls`
  focus task (block out an FK control-rig pose beat). Agent called
  `sequencer.get_binding_transform {binding:"EB7E779D449399A38C3A8DA6CA0FC15B", frame:0}`
  and got `[MISSING_REQUIRED_PARAM] Missing required parameter 'bindingId' (type: string)`
  after using the SAME `binding` key successfully on `add_controlrig_track` in the
  same run — one wasted round-trip from a cross-method binding-param naming
  inconsistency. CallAnalyzer flagged it as inefficiency `frustrating` on
  `sequencer.get_binding_transform`; agent friction note called it a "param slip
  (binding vs bindingId)". Companion process finding to the same task's tool bugs
  (`B-sequencer-add-actor-unbound-possessable`, `B-sequencer-create-save-no-disk-write`,
  filed by the per-finding judge). Sibling of the `param-alias` family, here as a
  cross-verb drift inside the `sequencer` namespace.
