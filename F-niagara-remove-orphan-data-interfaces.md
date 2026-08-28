---
id: F-niagara-remove-orphan-data-interfaces
title: "No verb can enumerate or remove the orphan resolved data interfaces that put a system into the VectorVM-assert state"
status: IN-REVIEW
severity: Medium
category: feature
tags: [niagara, data-interface, repair, orphan, template-leftover, recovery]
encounters: 1
lastSeen: 2026-08-27
---

# There is a refusal, and no way to act on it

When `niagara.add_emitter` / `remove_emitter` refuse with `NIAGARA_DATA_INTERFACE_MISMATCH`, the
documented recovery is: compile the system and retry; if it still refuses, the resolved set carries
data interfaces the compiled scripts no longer reference -- typically leftovers from an emitter
duplicated off a stock template -- and they must be removed from the emitter.

There is no verb that removes them, and none that even lists them. The error payload names the
offending scripts and both counts, which is enough to know you are stuck and not enough to get unstuck.
The only route out today is rebuilding the system from a stock one with `asset.duplicate`.

**Fix:** a verb that enumerates resolved data interfaces with no compiled counterpart, and one that
removes them. The enumeration half is close to free -- `CheckDataInterfaceCounts` already walks both
sets to produce the count.

## History
- `#1-recovery-gap-named-by-the-di-fix` `OPEN` reporter -- Listed by the agent that added the
  data-interface gate, as one of the parent ticket's fix directions it could not take from inside one
  handler file.
- `#2-enumerate-and-remove-orphans` `IN-REVIEW` developer — "Added `FindOrphanResolvedDataInterfaces` / `RemoveOrphanResolvedDataInterfaces` in NiagaraDataInterfaceConsistency.cpp matching each resolved entry's `CompileName` against the script's compiled `DataInterfaceInfo`, and registered `niagara.list_orphan_data_interfaces` / `niagara.remove_orphan_data_interfaces` in NiagaraAdvancedEditHandler.cpp; removal is planned whole-system first and refused with `NIAGARA_ORPHAN_REMOVAL_UNSAFE` (nothing mutated) unless dropping the orphans reconciles every script's two counts, then prunes both the resolved set and the cached defaults it is rebuilt from, remaps the resolved user-DI binding indices, runs `OnCompiledDataInterfaceChanged`, and re-measures the verdict"
