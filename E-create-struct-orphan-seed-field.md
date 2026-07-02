---
id: E-create-struct-orphan-seed-field
title: "blueprint.create_struct with no fields leaves a stray default MemberVar_0; the add_struct_field workflow never reclaims it"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [blueprint, user-defined-struct, create_struct, add_struct_field, seed-field, member-var-0, docs]
---

# `blueprint.create_struct` leaves an orphan seed field on the empty-create + add path

`FStructureEditorUtils::CreateUserDefinedStruct` always seeds a new struct with
one default bool member (`MemberVar_0`) — UE requires a user-defined struct to be
non-empty, so the asset can never start with zero fields. The `create_struct`
handler already knows this and absorbs the seed: when the caller passes a `fields`
array at creation, the first seeded field reuses the auto-created member (the
`bSeedFieldReused` branch — `BlueprintTypeDefinitionHandler.cpp` `create_struct`
handler, `ExistingVars.Num() == 1` → `UpdateStructField(... SeedGuid ...)`).

The gap is the **other** common workflow: create the struct empty, then add each
field with separate `blueprint.add_struct_field` calls. That path never reclaims
the seed. `add_struct_field` always calls `FStructureEditorUtils::AddVariable`
(it has no equivalent of the create-time `bSeedFieldReused` reuse), so the struct
ends up with `MemberVar_0` **plus** every added field. A caller who adds N fields
intending N fields gets N+1, with a phantom bool field they never asked for. To
get the intended layout they must notice the extra field, `list_struct_fields` to
read it, and `remove_struct_field` it. (The untouched seed's friendly name *is*
its internal name `MemberVar_0`, so removing the seed by that name works under the
old code — the GUID-mangled-name removal pain of
`B-remove-struct-field-friendly-name-rejected` bites the broader rename-then-remove
workflow, not the seed itself.)

The schema text is **silent** about the seed rather than affirmatively wrong:
`create_struct`'s `fields` param doc reads *"Initial field array of {name, type}
entries; type tokens match blueprint.add_variable's variableType."* — it never
mentions `MemberVar_0`, so omitting `fields` is reasonably read as "empty struct"
when it actually leaves the seed behind. (The "Empty/omitted leaves the … without
entries" wording belongs to `create_enum`'s `entries` param, not here.)

## What it should do

Pick one (the first is the clean ergonomic fix; the second is the doc floor):

1. **Reclaim the seed on first add.** Have `blueprint.add_struct_field` reuse the
   lone auto-seeded member the same way `create_struct`'s seed loop does: if the
   struct currently has exactly one field and it is the untouched default seed
   (single field named `MemberVar_0` / engine default-name shape, bool, no
   user-set tooltip/metadata/default), rename+retype it instead of appending.
   That makes "create empty, then add fields" produce exactly the fields added,
   matching the create-with-`fields` path and caller intent.
   - Alternatively, drop the seed inside `create_struct` itself after creation
     when no `fields` are supplied, so an "empty" struct really is empty — but UE
     forbids a zero-field struct, so seed-reclamation on first add is the safer
     route than persisting a truly empty asset.

2. **At minimum, document it.** The `create_struct` `fields` param doc is silent
   about the seed — add a note that the engine seeds a default `MemberVar_0` field
   (UE forbids a zero-field struct) and that the first `fields`/`add_struct_field`
   entry reclaims it, and add a `create_struct` section to the
   `docs/wiki-src/blueprint.md` overlay (which currently has none) calling out the
   seed and the "seed via `fields` at creation, or let the first `add_struct_field`
   reclaim it" guidance. (Downstream wiki edit — overlay page:
   `docs/wiki-src/blueprint.md`, `blueprint.create_struct` section.)

**Workaround:** pass all fields in the `create_struct` `fields` array at creation
(the seed is then reused into the first field, no orphan) — or, after building via
`add_struct_field`, `list_struct_fields` and `remove_struct_field` the leftover
`MemberVar_0` (the untouched seed's name == `MemberVar_0`, so removal by that name
works).

## Evidence (this task's call-log)

Story: create `/Game/Data/S_InventoryItem` empty, then five `add_struct_field`
calls (ItemName, StackCount, Weight, IsConsumable, MaxStack). Friction note:

> create_struct silently seeds a default bool field "MemberVar_0" so an "empty"
> create actually starts with 1 field, leaving 6 instead of 5 — had to detect and
> remove that stray field too.

Call-log cost: after the field adds, the agent issued an extra
`blueprint.list_struct_fields` (diagnose) and an extra
`blueprint.remove_struct_field {stray MemberVar_0 by internal name}` purely to
clean up the seed — 2 calls and one round-trip that the create-with-`fields`
path would not have needed. (Removing the *seed* itself is unaffected by
`B-remove-struct-field-friendly-name-rejected`: the untouched seed's friendly name
is its internal name `MemberVar_0`, so removal works either way. The GUID-mangled
removal pain bites the broader rename-then-remove workflow, not the seed.) Source
confirmation: `create_struct` reuses the seed only behind `bSeedFieldReused` (gated
on a non-empty `fields` array); `blueprint.add_struct_field` has no seed-reuse
branch and always `AddVariable`s.

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of task focus `blueprint.list_struct_fields`. `blueprint.create_struct` with no `fields` leaves the engine's default `MemberVar_0` seed member in place; the "create empty then `add_struct_field`" workflow never reclaims it (only the create-time `fields` seed loop does, via `bSeedFieldReused`), so N adds yield N+1 fields and the caller must detect + remove the orphan (extra `list_struct_fields` + `remove_struct_field`, the latter blocked on `B-remove-struct-field-friendly-name-rejected`). Schema doc ("Empty/omitted leaves the struct without fields") is misleading. Distinct PROCESS angle from the judge-filed friendly-name removal bug. Fix: reclaim the lone untouched seed field on first `add_struct_field` (mirror create_struct's seed reuse), or at minimum document the seed in the param doc and `docs/wiki-src/blueprint.md` create_struct overlay.
- `#2-reword-fix` `IN-REVIEW` developer — Reworded two inaccurate evidence points (all three validity lenses flagged them): (a) the "misleading schema text" claim — `create_struct`'s `fields` doc does NOT say "Empty/omitted leaves the struct without fields"; that wording is `create_enum`'s `entries` param. Restated as the doc being *silent* about the seed (`BlueprintTypeDefinitionHandler.cpp:904` read "Initial field array of {name, type} entries…"), not affirmatively wrong. (b) Softened the `B-remove-struct-field-friendly-name-rejected` coupling: the untouched seed's friendly name == internal name `MemberVar_0`, so removing the seed works under old code; the GUID-mangle removal pain bites the rename-then-remove workflow, not the seed. Implemented Fix #1 + Fix #2. **Fix #1 (root-cause ergonomic fix):** added `IsUntouchedSeedField()` (single field, `MemberVar_<n>` default-name shape, plain bool, no container, no user-set default/tooltip/metadata) and a seed-reuse branch in `AddStructField` that rename/retypes the pristine engine seed in place on the first add (then applies rich optionals + notify) instead of appending — mirrors `create_struct`'s `bSeedFieldReused`, so "create empty + N add_struct_field" yields exactly N fields. The strict guard never overwrites a user-edited lone field. **Fix #2 (docs):** extended the `create_struct` `fields` param doc to describe the `MemberVar_0` seed + reclamation, and added a `### blueprint.create_struct` section to the `docs/wiki-src/blueprint.md` overlay (previously had none). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BlueprintTypeDefinitionHandler.cpp` (handler + param doc), `docs/wiki-src/blueprint.md` (wiki overlay). Test: `FBlueprintAddStructFieldReclaimsSeedTest` (`EditorAutomationRpcGateway.blueprint.add_struct_field.ReclaimsOrphanSeed`) in `Source/EditorAutomationRpcGateway/Private/Tests/Blueprint/TestBlueprintHandlers.cpp` — creates an empty struct, adds 3 fields one at a time, asserts `count == 3` and that no `MemberVar_0` remains; fails (count 4 + leftover seed) if the seed-reuse branch is reverted.
