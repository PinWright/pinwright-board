---
id: E-niagara-module-input-default-value-unreported
title: "DUPLICATE of B-inspect-default-module-input-has-no-effective-value — an unset module input's effective value is unreadable"
status: DUPLICATE
severity: Medium
category: bug
tags: [duplicate, niagara, inspect, module-input, defaults]
encounters: 1
lastSeen: 2026-09-05
duplicateOf: B-inspect-default-module-input-has-no-effective-value
---

# Merged — see `B-inspect-default-module-input-has-no-effective-value`

This ticket and `B-inspect-default-module-input-has-no-effective-value` were filed independently,
minutes apart, by two agents hitting the same defect on different assets after the 2026-09-05
plugin rebuild. **The other ticket is canonical.** Work it there; nothing is tracked here.

The file is kept rather than deleted because the board is shared by several hosts and a vanished
id reads worse than a stub.

Everything this ticket contributed has been folded into the canonical ticket and credited to its
History `#2`:

- the **`PersistentGuid` vs `PinId` root cause** — `UNiagaraNodeParameterMapGet::PinOutputToPinDefaultPersistentId`
  is keyed on each pin's `PersistentGuid` while `inspect` publishes its `PinId`, so the two id
  spaces never meet and the linkage is serialized under an identifier the API does not expose;
- the **`NiagaraNodeParameterMapGet_0`** observation that its pin order is neither forward nor
  reversed, alongside the `NiagaraNodeParameterMapGet_6` reversed-order counter-example both
  reporters found independently;
- the **working read route** — `property.get` on the graph's `VariableToScriptVariable`, then
  `property.list` on the resolved `UNiagaraScriptVariable`, decoding `VarData` by hand — with the
  three measured values (`Module.Cone Axis` = `(1,0,0)`, `Cone Axis Coordinate Space` = `2` =
  `Local`, `Local.Module.ConeVector` = `(0,0,0)`);
- the **`## Fix`** in full, including sourcing the default from `UNiagaraScriptVariable` rather
  than from the unpairable MapGet pins, and the `niagara_stack.json` aspect-version bump;
- the **`NS_Blood` outcome** — Drips / Spray / Mist were accidentally correct and are now stated
  explicitly.

The classification moved from `ergonomic` to `bug` on the merge: an unreadable value is not a
convenience gap, it made a live contract question undecidable for two builds.

## History
- `#1-default-unreadable` `OPEN` reporter — Original report. Hit while stating `Cone Axis` / `Cone Axis Coordinate Space` explicitly on `/Game/FPS/VFX/NS_Blood`'s Drips/Spray/Mist emitters, 2026-09-05, UE 5.8, EAContentExamples58, live editor on port 27145, plugin freshly pulled to `origin/master` and rebuilt. Full text, evidence and non-confirmations are preserved in `B-inspect-default-module-input-has-no-effective-value` History `#2`.
- `#2-merged-as-duplicate` `DUPLICATE` VFX — Reduced to this stub on the coordinator's instruction after both tickets were found on the board minutes apart. `B-inspect-default-module-input-has-no-effective-value` is canonical and carries every finding from `#1`, attributed. No content was dropped in the merge. Nothing further should be appended here; append to the canonical ticket instead.
