---
id: F-gas-configure-asc-bpir
title: "Reimplement gas.configure_asc by emitting SetReplicationMode into the Blueprint graph via BPIR (removed: the template write never persisted)"
status: OPEN
severity: Medium
category: feature
tags: [gas, configure_asc, replication, ability-system, bpir, reimplement, rpc-audit]
---

# Reimplement `gas.configure_asc` via BPIR (graph call), not a component-template property write

`gas.configure_asc` was **removed** in the batch-2 RPC audit recorded in
[`E-rpc-audit-43-record`](E-rpc-audit-43-record.md): its write target was wrong
at the data-model level, so nothing it did ever persisted. Setting an ASC's
gameplay-effect replication mode (`full` / `mixed` / `minimal`) on a Blueprint is
still a legitimately wanted capability, and it is the first thing any GAS setup
has to get right, so it should come back the way the engine actually expects it
to be done.

## What the removed version did wrong

The handler resolved the Blueprint's `AbilitySystemComponent` template and called
`SetReplicationMode(...)` on it. The mode is **not serialized on the component
template**, so the value lived only in memory for the current editor session and
was gone on reload. The asset on disk never carried the requested mode.

There is no property-level fix here. `property.set` cannot help either: there is
no serialized property on the template to write. Any fix that stays at the
template level has the same defect.

## Conflict with the earlier ticket (worth reading before you believe an "it round-trips" claim)

[`E-configure-asc-echoes-invalid-replication-mode`](E-configure-asc-echoes-invalid-replication-mode.md)
(now closed `WONTFIX`, see its `#3`) reported that a *valid* mode "works and
round-trips correctly (verified live: `mixed`->`mixed`, `minimal`->`minimal` via
`gas.get_gas_info`)". That observation is real but proves nothing: `get_gas_info`
read back the **same in-memory component template** the write had just touched, in
the same session. The apparent round-trip is the write and the read agreeing with
each other inside a single process lifetime, not the value reaching the asset.
Any reimplementation must be verified across a **save + reload**, not by an
immediate readback.

## Proper implementation: emit the call into the graph via BPIR

Shipping games set this in code, at runtime, on the owning actor:
`ASC->SetReplicationMode(EGameplayEffectReplicationMode::Mixed)`. The Blueprint
equivalent is a `SetReplicationMode` call node against the ASC component,
executed on a construction/begin-play path. PinWright already has the machinery
to author exactly that: the **BPIR compiler**
(`blueprint.compile_bpir` / `blueprint.insert_bpir_at_node`).

The reimplemented `gas.configure_asc` should therefore:

- Resolve the ASC component on the target Blueprint (fail loud with a clear error
  if the Blueprint has no ASC, rather than creating one silently).
- Emit a `SetReplicationMode` call node into the graph, target = the ASC
  component reference, with the `EGameplayEffectReplicationMode` enum literal for
  the requested mode, wired onto an appropriate exec path (BeginPlay is the
  obvious default; the exact insertion point is a design decision the ticket
  should settle before implementation, see the open question below).
- Be **idempotent**: a second call with a different mode must update the existing
  node's enum literal, not append a second `SetReplicationMode` call. This is the
  main thing that separates a BPIR emit from a property write and the main thing
  the test must pin.
- Validate the mode string up front and reject an unrecognized value with
  `INVALID_PARAMS` naming `full|mixed|minimal` (this was the fix that landed on
  the now-closed `E-configure-asc-echoes-invalid-replication-mode` and it should
  survive into the reimplementation, not be re-lost).

**Fix:** new handler emitting BPIR, in the GAS handler where the removed method
lived. Regression test: build a Character BP with an ASC, run `configure_asc` with
`mixed`, assert a `SetReplicationMode` node exists in the graph with the `Mixed`
literal, **save and reload the asset**, and assert the node is still there (the
removed template-write version leaves nothing behind after reload, which is the
differential proof). Second call with `minimal` must mutate the same node, not add
a second one.

## Open question

Which exec path should the emitted call sit on? BeginPlay is the simplest and
matches the common hand-written pattern, but ASC replication mode is usually set
on the server right after the ASC is initialized (for a Character, on
`PossessedBy` / `OnRep_PlayerState`). If the handler hardcodes BeginPlay it may be
wrong for the PlayerState-owned-ASC layout. Consider exposing the insertion point
as a parameter with BeginPlay as the default.

## History
- `#1-reimpl-after-audit` `OPEN` reporter - Filed to reinstate the wanted capability removed by the batch-2 RPC audit ([`E-rpc-audit-43-record`](E-rpc-audit-43-record.md)). The removed `gas.configure_asc` called `SetReplicationMode` on the ASC component template, where the mode is not serialized, so the write only ever existed in memory for the current session and never reached the asset. There is no property-level fix (`property.set` has no serialized property to target). Proper path: emit a `SetReplicationMode` call node into the Blueprint graph via PinWright's BPIR compiler (`blueprint.compile_bpir` / `blueprint.insert_bpir_at_node`), targeting the ASC component with the `EGameplayEffectReplicationMode` literal, idempotent on repeat calls, keeping the `INVALID_PARAMS` rejection of an unrecognized mode that had landed on the now-closed [`E-configure-asc-echoes-invalid-replication-mode`](E-configure-asc-echoes-invalid-replication-mode.md). Flagged the conflict with that ticket's "verified live, round-trips correctly" note: the `gas.get_gas_info` readback hit the same in-memory template the write had just touched, so it never proved persistence. Verification must be save + reload, and the open question of which exec path the emitted call belongs on (BeginPlay vs PossessedBy/OnRep_PlayerState) should be settled before implementation.
