---
id: B-niagara-create-payload-defaults
title: "niagara.graph.create_node accepts invalid class-specific payload values and silently keeps defaults"
status: OPEN
severity: Medium
category: bug
tags: [niagara, create-node, payload, validation, false-success, static-switch]
---

# Invalid nested values create a different node than requested

`ApplyCreateNodePayload` validates `opName`, but other class-specific values are permissive
(`NiagaraGraphHandler.cpp:470-620`):

- an unknown Input-node `usage` matches no branch and leaves the default usage;
- an unknown `staticSwitchType` matches no branch and leaves the default switch type;
- `staticSwitchType:"Enum"` with an unloadable `enumPath` leaves the enum null;
- `target.kind` is read by the outer handler and deliberately ignored while graph remains selected
  (`:667-671`).

All paths return an empty `FNiagaraEditError`, finalize the node, optionally compile/save, and send
success. A typo therefore changes node semantics or creates an incomplete enum switch while the
caller is told its requested payload was applied.

`payload.inputType` has the same code shape but is already tracked separately by
`E-niagara-create-node-input-type-unverifiable`; this ticket covers the remaining invalid-value
fallbacks rather than duplicating that field.

## What it should do

Validate each supplied discriminator against an explicit allow-list, require a loadable enum for an
Enum switch, and reject non-graph `target.kind`. Only omitted fields may retain defaults. Include
the effective usage/switch type/enum in response or readback tests.

## Workaround

Use only the exact supported values and re-read the node before trusting success.

## Related

- `E-niagara-create-node-input-type-unverifiable`
- `B-declared-param-guard-blind-to-nested-keys`

## History
- `#1-source-scan` `OPEN` reporter -- Full branch audit distinguished absent defaults from supplied
  invalid values; no node was created under the scan constraints.
