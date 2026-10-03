---
id: B-niagara-create-payload-defaults
title: "niagara.graph.create_node accepts invalid class-specific payload values and silently keeps defaults"
status: DONE
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
- `#2-validate-discriminators` `IN-REVIEW` developer -- Still reproducible at HEAD before the fix (the usage/staticSwitchType `if/else if` chains had no `else`, the Enum branch set the type before trying `LoadObject`, and the handler's `kind` branch was an empty block). `NiagaraGraphCreate::ApplyCreateNodePayload` (`Source/PinWright/Private/Handlers/Niagara/NiagaraGraphHandler.cpp`) now refuses an unknown Input `usage` and unknown `staticSwitchType` with `INVALID_ARGUMENT` naming the value and the allow-list; `staticSwitchType:"Enum"` requires `enumPath` (`INVALID_ARGUMENT`) that loads (`ENUM_NOT_FOUND`, quiet load) and only then sets type + enum; the create_node handler refuses a non-`graph` `target.kind` with `INVALID_TARGET_KIND` before resolving the asset. Omitted fields still keep defaults. Refusals inside the node-creator window reuse the existing remove-node + dirty-restore path. `payload.inputType` untouched (tracked by `E-niagara-create-node-input-type-unverifiable`). Effective usage / switch type / enum are asserted by readback. Test: `PinWright.niagara.graph.create_node.InvalidPayloadDiscriminatorsAreRefused` (new file `Tests/Niagara/TestNiagaraGraphCreateNodePayloadValidation.cpp`). Docs: `docs/wiki-src/niagara.graph.md` (create_node), CHANGELOG. Behaviour change: those inputs used to succeed.
- `#3-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit b39ceafe). run3/full passed non-skipped: `PinWright.niagara.graph.create_node.InvalidPayloadDiscriminatorsAreRefused`, plus the neighbours `.PayloadApplicationAndErrorCodes`, `.RefusalLeavesPackageClean` and `PinWright.niagara.graph.create_node_guard.RejectionLeavesGraphUnchanged`. Acceptance: an unknown Input `usage` and an unknown `staticSwitchType` are refused `INVALID_ARGUMENT` with the allow-list. `staticSwitchType:"Enum"` without `enumPath` is refused, and with an unloadable one it is refused `ENUM_NOT_FOUND`. A non-`graph` `target.kind` is refused `INVALID_TARGET_KIND` before the asset resolves. Omitted fields keep their defaults, and the effective usage, switch type and enum are asserted by readback. Refusals leave the graph and package clean. `payload.inputType` is out of scope (tracked by `E-niagara-create-node-input-type-unverifiable`). Behaviour change: these inputs used to succeed.
