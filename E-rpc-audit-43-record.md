---
id: E-rpc-audit-43-record
title: "RECORD: batch-2 RPC audit of 43 flagged methods (18 removed + 24 fixed in place + 1 kept)"
status: DONE
severity: Medium
category: ergonomic
tags: [rpc-cull, rpc-audit, removal, record, stub, superseded, fake-success, surface-area]
---

# RECORD: batch-2 RPC audit, 43 methods

This is an **append-only record/log entry**, not a fix ticket. It documents the
second cleanup pass over the PinWright RPC catalog (`Plugins/PinWright`), which
followed the 151-method cull recorded in
[`E-rpc-cull-151-record`](E-rpc-cull-151-record.md). Like that record it is filed
directly at `DONE` because it records completed work; it is outside the
OPEN -> IN-REVIEW -> DONE fix lifecycle (there is nothing to verify-then-close,
the audit is the fact being recorded). It exists so that anyone later asking
"what happened to `X`?" or "why does `Y` behave differently now?" has one
authoritative inventory with per-method replacements and per-method defect notes.

## Method

A 66-agent Opus audit swept the surviving catalog against a detection checklist:
**fake-success stubs** (report success, mutate nothing or mutate the wrong
thing), **clearly-broken methods** (the implementation does not do what the name
and docs say), **tombstones** (guarded NOT_IMPLEMENTED leftovers), and
**superseded methods** (a newer PinWright verb fully covers them). It flagged
**43** methods.

Disposition: **18 removed, 24 fixed in place, 1 kept.** The first batch removed
everything it flagged; this one did not, because most of batch 2's findings were
real methods with a specific, repairable defect (a dropped parameter, a wrong
write target, a bogus version guard) rather than hollow shells.

## Count note

The fixed-in-place inventory below enumerates **25 distinct method names** against
the audit's headline count of **24**. Both are right: 24 of them were among the 43
methods the audit flagged (18 + 24 + 1 = 43), while `pipeline.get_status` was not
an audit finding at all. It was a carried-over follow-up from batch 1, where it was
caught reporting hardcoded fabricated counters (1069 actions / 35 categories) and a
hardcoded "1.0.0" version, and it was fixed in the same wave for convenience.

## Removed (18)

### Superseded (9), `method -> replacement`

- `networking.set_net_role` -> `misc.set_replication` (see **Kept** below)
- `misc.set_net_update_frequency` -> `networking.configure_net_update_frequency`
- `misc.create_rpc` -> `networking.create_rpc_function`
- `misc.configure_net_cull_distance` -> `networking.configure_net_cull_distance`
- `misc.create_replicated_variable` -> `blueprint.add_variable` (`isReplicated`)
- `misc.create_spline_component` -> `blueprint.scs.add_component`
- `interaction.create_interactable_interface` -> `blueprint.create` (`blueprintType=interface`)
- `pipeline.run_ubt` -> `system.run_ubt`
- `data_table.describe` -> `data_table.list_rows` (byte-identical output)

### Fake or inert (4)

- `networking.configure_push_model` - wrote metadata that nothing reads.
- `networking.configure_replication_graph` - echoed dead params and wrote an unrelated flag.
- `performance.configure_world_partition` - targeted cvars that do not exist; always errored.
- `texture.create_ao_from_mesh` - never loaded the mesh. It produced a fake vignette, not ambient occlusion.

### Broken or destructive (5)

- `geometry.loop_cut` - not a loop cut at all: a destructive plane cut through the mesh. Real edge-loop insertion is genuinely missing, reimplementation tracked in [`F-geometry-loop-cut-proper`](F-geometry-loop-cut-proper.md).
- `geometry.chamfer` - a duplicate of `geometry.bevel` with a dead `steps` param. `bevel` was fixed instead (see below); the duplicate went.
- `animation.authoring.set_bone_key` - ignored its `frame` param and wiped the entire bone track on every call. Reimplementation tracked in [`F-anim-set-bone-key-proper`](F-anim-set-bone-key-proper.md).
- `gas.configure_asc` - wrote the ASC replication mode to a non-serialized component template, so the write never persisted. There is no property-level fix; reimplementation via BPIR tracked in [`F-gas-configure-asc-bpir`](F-gas-configure-asc-bpir.md).
- `ai.configure_state_tree_task` - dead `taskType` param. The real capability is `state_tree.add_task`.

## Fixed in place (24 audit findings + `pipeline.get_status`, see Count note)

- `networking.set_property_replicated` - now writes `NewVariables` + `CPF_Net` and compiles.
- `geometry.bevel`, `geometry.bridge` - dropped a bogus `<5.5` **upper** version guard that dead-ended the feature on current engines.
- `geometry.create_procedural_mesh` - real `SetCollisionEnabled` (it previously only called `SetGenerateOverlapEvents`).
- `spline.set_spline_point_tangents` - applies **both** the arrive and leave tangents. It previously claimed UE has only one tangent per point, which is false.
- `lighting.set_exposure` - sets the `bOverride_AutoExposure*` bits, without which the values it wrote were ignored.
- `material.authoring.set_two_sided` - added `PostEditChange` + save.
- `animation.authoring.add_slot_node` - bare `SlotName` + skeleton `SetSlotGroupName`.
- `animation.authoring.set_interpolation_settings` - applies `interpolationType`; only touches speed when speed is supplied.
- `skeleton.configure_physics_body` - `SetMassOverride(mass, true)`.
- `sequencer.add_camera_track` - binds the **named** camera; returns `CAMERA_NOT_BOUND` otherwise.
- `sequencer.remove_actors` - `RemoveSpawnable` fallback; reports success only when something was actually removed.
- `asset.get_dependencies_classified` - visited-set BFS. The prior recursion missed transitive deps.
- `asset.find_objects_by_tag` - dropped a dead `searchAssets` param.
- `ai.create_nav_modifier` - echoes the resolved area class instead of a hardcoded `"Default"`.
- `editor.eject` - `RequestToggleBetweenPIEandSIE` + verified state.
- `editor.possess` - direct `PC->Possess` instead of a nonexistent `"POSSESS"` exec string.
- `effect.advance_simulation` - a single `AdvanceSimulation` call. It previously advanced `Steps^2` frames.
- `gas.add_tag_to_asset` - 4 of its 5 branches were real; the Actor-with-ASC branch fabricated a blank, unread `OwnedGameplayTags` variable and reported false success. That branch is now rejected with `UNSUPPORTED_TYPE`.
- `level.structure.create_level_instance` - `SetWorldAsset` + update. The level asset param was previously ignored.
- `level.structure.create_packed_level_actor` - honors the pack flags via `FPackedLevelActorBuilder`.
- `level.structure.configure_hlod_layer` - actually writes `CellSize` / `LoadingRange`.
- `level.structure.configure_level_streaming` - applies `streamingMethod`.
- `level.build_lighting` - quality plumbed through `FLightingBuildOptions` instead of an exec string.
- `pipeline.get_status` - had been reporting **hardcoded fabricated** counters (1069 actions / 35 categories) and a hardcoded `"1.0.0"` version. It now reads the live handler registry and the plugin descriptor.

## Kept (1)

`misc.set_replication`. It and `networking.set_net_role` each cited the other as
their replacement, a supersession cycle that had to be broken by hand.
`set_net_role`'s role semantics were **fake** (every role collapsed to a single
bool), while `set_replication` honestly calls `SetReplicates` +
`SetReplicateMovement`. So the honest one survives as the only direct verb for
`bReplicates`, and `set_net_role` is in the removal list above.

## Tickets filed out of this audit

Three removals took a **legitimately-wanted capability** with them (the impl was
fake or destructive, the idea was not). Proper reimplementations are filed as
OPEN feature tickets:

- [`F-geometry-loop-cut-proper`](F-geometry-loop-cut-proper.md) - real edge-loop insertion.
- [`F-anim-set-bone-key-proper`](F-anim-set-bone-key-proper.md) - per-frame bone-key splice (`add_bone_track` and bone-track readback both survive, so this is a write verb only).
- [`F-gas-configure-asc-bpir`](F-gas-configure-asc-bpir.md) - ASC replication mode emitted into the graph via the BPIR compiler.

One defect was found during the audit and **deliberately left unfixed**, filed as
its own OPEN bug:

- [`B-bridge-subdivisions-ignored`](B-bridge-subdivisions-ignored.md) - `geometry.bridge` reads and echoes a `subdivisions` param it never applies. Same dead-knob class as the original `bevel` finding, and it survives the `bridge` fix that landed in this audit.

## Ticket orphaned by a removal

[`E-configure-asc-echoes-invalid-replication-mode`](E-configure-asc-echoes-invalid-replication-mode.md)
was `IN-REVIEW` with a landed `INVALID_PARAMS` validation fix for
`gas.configure_asc`. That method is now removed outright, so the fix and its
regression test went with it and there is nothing left to verify. The ticket has
been closed `WONTFIX` (closed by removal, not by refusal) and cross-references
[`F-gas-configure-asc-bpir`](F-gas-configure-asc-bpir.md), which carries the
validation requirement forward.

## History
- `#1-audit-record` `DONE` reporter - Record entry (filed directly at DONE; a record, not a fix-lifecycle ticket). Documents the batch-2 RPC audit that followed the 151-method cull ([`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)): a 66-agent Opus sweep against a fake-success / clearly-broken / tombstone / superseded detection checklist flagged 43 methods, dispositioned 18 removed, 24 fixed in place, 1 kept. Full per-method inventory pasted above: 18 removals (9 superseded with replacements, 4 fake-or-inert, 5 broken-or-destructive), 24 in-place fixes with a one-line defect note each (the enumeration carries 25 method names against the headline 24, flagged in the Count note rather than reconciled), and the single kept method (`misc.set_replication`, which survived a mutual-supersession cycle with the fake-semantics `networking.set_net_role`). Three legitimately-wanted capabilities whose implementations were fake or destructive were re-filed as OPEN reimplementation tickets (F-geometry-loop-cut-proper, F-anim-set-bone-key-proper, F-gas-configure-asc-bpir), and one deliberately-unfixed audit finding was filed as B-bridge-subdivisions-ignored. E-configure-asc-echoes-invalid-replication-mode, orphaned by the `gas.configure_asc` removal, was closed WONTFIX with a cross-reference to the BPIR reimplementation ticket.
