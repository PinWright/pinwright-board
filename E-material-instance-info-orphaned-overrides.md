---
id: E-material-instance-info-orphaned-overrides
title: "material.authoring.get_material_instance_info mixes orphaned overrides into overrides[] with nothing marking them dead — eight parameters the parent no longer declares read exactly like live ones"
status: OPEN
severity: Medium
category: ergonomic
tags: [material, material-authoring, get_material_instance_info, material-instance, overrides, orphaned-parameters, readback, asset-dump, weapons]
encounters: 3
lastSeen: 2026-09-06T00:00:00Z
---

# An override for a parameter the parent deleted reads identically to one that drives the shader

When a parent material drops a parameter, the instances that overrode it keep the override in their
`ScalarParameterValues` / `VectorParameterValues` arrays. Nothing prunes it, and nothing renders
with it — it is dead weight that still serialises. `get_material_instance_info` reports it in
`overrides` beside the live entries, in the same shape, with no flag.

## What was called

```
material.authoring.get_material_instance_info {assetPath: "/Game/FPS/Weapons/Materials/MI_WPN_*"}
```

on all six `MI_WPN_*` instances.

## What happened — measured

Every one of the six returned **8 scalar/vector overrides for parameters the parent material no
longer declares**:

`WearStrength`, `SmudgeStrength`, `DustStrength`, `MicroGrainStrength`, `MicroGrainFreq`,
`WearFreq`, `SmudgeFreq`, `DustTint`

They arrive inside `overrides` mixed among the live overrides, indistinguishable from them.

Confirmed on two independent surfaces, not just the RPC:

- **Byte-confirmed present in all four weapon MI `.uasset`s** — the names are in the package bytes,
  so this is the on-disk state, not a readback artefact.
- **`M_WPN_Master` declares 18 live parameters and none of those 8.** So all eight are orphans;
  none is a name-case or group mismatch.

The only way to tell a live override from a dead one is to fetch the parent's `parameters[]`
separately and hand-diff the two lists by name — per instance, per parameter type.

## What was expected

That a readback which is explicitly the "did my write land / what is on this instance" surface
distinguishes an override that affects the render from one that does not. The information needed
to make that call is already in the same handler's hands: `get_material_instance_info` **already
returns `parameters[]`** (the parent's declared set, with `type`/`group`/`sortPriority`) in the same
response as `overrides`. The comparison is one set-membership test per override, on data already
loaded.

## The failure this actually caused

A round-1 defect was closed on the strength of "`WearStrength` is non-zero now" — read straight off
`overrides` in a `get_material_instance_info` response. `WearStrength` is one of the eight dead
parameters. The override was written, the readback confirmed it, and the shader never saw it: a
non-zero value on a parameter the parent does not declare changes nothing. **The verb's own output
was the evidence for a fix that was not a fix.**

This is the trap the whole `F-material-instance-overrides-incomplete` read-back work exists to close
— that ticket's stated purpose for `get_material_instance_info` is that "a caller cannot verify that
`set_scalar_parameter_value` actually took effect". It closed the *missing readback* half. An
override that reads back correctly and still does nothing is the same failure wearing the fix.

## What is asked for

**Mark each override live or dead against the parent's declared set.** Smallest sufficient change:
add `live: true|false` to each entry in `overrides` (or, if the entry shape must stay flat for
compatibility, an `orphanedOverrides: {scalar: [...], vector: [...], texture: [...], staticSwitch: [...]}`
sibling block listing the names). Either shape lets a caller assert on the field instead of
diffing two lists.

Two follow-ons that cost almost nothing once the comparison exists:

- **Same field on `asset.dump`'s `material_instance.json`**, which the docs describe as mirroring
  this payload — otherwise the dump sidecar keeps the trap the RPC drops (`E-dump-rpc-parity`).
- **A count in the summary line** (`orphanedOverrideCount: 8`) so a folder sweep can find the
  instances worth cleaning without reading every response.

Remediation already exists once they can be *found*: `material.authoring.clear_parameter_override`
(shipped under `F-material-instance-overrides-incomplete` `#3`) removes them one at a time. The gap
is purely detection — the cleanup verb is unusable because nothing tells you what to point it at.

## Root cause — guess, no source read taken

The override serialiser almost certainly walks the instance's own
`ScalarParameterValues` / `VectorParameterValues` arrays and emits every entry, without consulting
the parent's declared parameter set that the same response builds for `parameters[]`. **This is an
inference from the response shape; no plugin source was opened for this ticket and no `file:line`
is claimed.**

## Severity

**Medium** — the rubric's soft-blocker band. The answer is obtainable (fetch the parent's
`parameters[]`, hand-diff by name, per type, per instance), so it is a documented-workaround case
rather than a hard blocker, which keeps it below High. But note it sits on the boundary: the
workaround is one a caller only runs *if they already suspect* orphans, and the measured cost here
was a wrong "fixed" verdict accepted from the verb's own output. Reach is not adjusted — material
instances are where all instance-side authoring verification lands, but this specific readback is
not an every-session call.

## Related

- `F-material-instance-overrides-incomplete` (IN-REVIEW, High) — **the ticket that created
  `get_material_instance_info`.** Its gap #4 was "no way to read instance values back"; this is the
  follow-on that the readback, as shipped, cannot distinguish live overrides from dead ones. Its
  `clear_parameter_override` (gap #1, shipped in `#3`) is the remediation this detection would
  finally make usable. Same handler, same response builder.
- `B-asset-dump-mic-no-material-instance-json` — the dump-side sibling surface that the docs say
  mirrors this payload; whatever flag lands here must land there too.
- `E-get-material-info-no-param-defaults` (IN-REVIEW) — the parent-side readback that omits parameter
  defaults. A caller doing the hand-diff workaround above is reading exactly that response, so the
  two gaps compound.
- `B-asset-dump-properties-spurious-override-on-bp-internal-bools` — the same category of defect on a
  different surface: an override reported where none meaningfully exists.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 2. `material.authoring.get_material_instance_info` on all six `MI_WPN_*` instances returned 8 scalar/vector overrides for parameters the parent material no longer declares — `WearStrength`, `SmudgeStrength`, `DustStrength`, `MicroGrainStrength`, `MicroGrainFreq`, `WearFreq`, `SmudgeFreq`, `DustTint` — mixed into `overrides` in the same shape as the live entries with nothing to tell them apart. Corroborated on two surfaces: the eight names are byte-confirmed present in all four weapon MI `.uasset`s (so this is on-disk state, not a readback artefact), and `M_WPN_Master` declares only 18 live parameters, none of which is one of the eight (so all eight are genuine orphans, not case or group mismatches). The only way to tell live from dead is to fetch the parent's `parameters[]` and hand-diff by name, per type, per instance — even though the same response already carries `parameters[]`, so the set-membership test is over data the handler has already loaded. Measured cost: a round-1 defect was closed on "`WearStrength` is non-zero now", read straight off this verb's `overrides` — `WearStrength` is one of the eight dead parameters, so the write landed, the readback confirmed it, and the shader never saw it; the verb's own output was the evidence for a non-fix. Ask: mark each override `live: true|false` against the parent's declared set (or emit a sibling `orphanedOverrides` block), mirror the field onto `asset.dump`'s `material_instance.json` per `E-dump-rpc-parity`, and add an `orphanedOverrideCount` so a folder sweep can find instances worth cleaning; the cleanup verb `material.authoring.clear_parameter_override` already exists and is unusable only because nothing says what to point it at. Root cause is a **guess**: the serialiser likely walks the instance's own `ScalarParameterValues`/`VectorParameterValues` and emits every entry without consulting the parent set the same response builds for `parameters[]` — inferred from the response shape, no plugin source was opened and no `file:line` is claimed. Severity Medium on the soft-blocker band (a hand-diff workaround exists), noting it borders High because the workaround is only run by a caller who already suspects orphans.
- `#2-still-open-six-instances-48-dead-overrides` `OPEN` WEAPONS-critic — Re-measured in a WEAPONS critic review round 3: **nothing has changed, and the count is bigger than `#1` recorded.** All **six** `MI_WPN_*` instances still override the same eight parameters the master no longer declares — `MicroGrainStrength`, `WearStrength`, `SmudgeStrength`, `DustStrength`, `MicroGrainFreq`, `WearFreq`, `SmudgeFreq`, `DustTint` — which is **48 dead overrides** across the kit (`#1` byte-confirmed the names in four `.uasset`s; all six carry them). The master now declares **23** parameters, up from the 18 `#1` measured, and **none of the eight is among them**, so the parent has been edited since `#1` without the orphans being pruned or becoming live again — the set is stable, not transitional. `material.authoring.get_material_instance_info` still returns `overrides` and `parameters` as **independent blocks with no cross-check**: no `orphaned` field, no `live` flag, no `orphanedOverrideCount`, and no warning, even though the same response carries both sides of the set-membership test. The verb has been asked the question twice now, on the same assets, and answered identically both times, so this is reach evidence rather than a new defect: `encounters` 1 → 2, **severity unchanged at Medium**. The measured cost from `#1` — a round-1 defect closed on "`WearStrength` is non-zero now", read straight off this verb's `overrides`, where the write landed, the readback confirmed it, and the shader never saw it — stands as the reason this is worth more than a docs note. `material.authoring.clear_parameter_override` still exists and is still unusable for lack of anything saying what to point it at. No plugin source was opened for this entry.
- `#3-unchanged-at-48-and-the-rot-is-bounded` `OPEN` WEAPONS-critic — Re-measured in a WEAPONS critic review round 4: **unchanged at 48 dead overrides**, byte-verified again this round. The same six weapon MIs each still override the same eight parameters `M_WPN_Master` no longer declares — `MicroGrainStrength`, `WearStrength`, `SmudgeStrength`, `DustStrength`, `MicroGrainFreq`, `WearFreq`, `SmudgeFreq`, `DustTint` — 6 x 8 = 48, the figure `#2` established, holding across a third consecutive round. `material.authoring.get_material_instance_info` still returns all of them verbatim under `overrides.scalar` / `overrides.vector`, the parent's `parameters[]` in the same response still omits all eight, and there is still **no `orphaned` field, no `live` flag, no `orphanedOverrideCount` and no warning anywhere** — so the set-membership test the response already carries both sides of is still not performed. **New this round, and it narrows the fix's blast radius:** the newer instances are clean. `MI_WPN_OpticLens` and the four `MI_VM_WPN_*` carry **zero** orphaned overrides. So the rot is confined to the six originals and is not being reproduced by whatever authors new instances — this is a one-time inheritance from the master's parameter set changing under existing children, not an ongoing defect in the authoring path. Two consequences worth recording for a fixer: the detection ask is unchanged in shape but its expected output is now known to be sparse (5 of 11 instances report nothing), which makes an `orphanedOverrideCount` in the summary line the highest-value part of the ask — a folder sweep would point straight at the six that need `clear_parameter_override` and skip the five that do not; and a fix can be validated against a clean control in the same kit rather than needing a synthetic fixture. `encounters` 2 → 3, **severity unchanged at Medium**, **status unchanged at OPEN**. The measured cost from `#1` — a round-1 defect closed on "`WearStrength` is non-zero now", read straight off this verb's `overrides`, where the write landed, the readback confirmed it and the shader never saw it — still stands as the reason this outranks a docs note. No plugin source was opened for this entry.
- `#4-subject-has-disappeared-needs-a-synthetic-fixture` `OPEN` WEAPONS-critic — **The motivating case is gone from this project, and with it the ability to test the verb here.** Status unchanged at `OPEN`; `encounters` / `lastSeen` deliberately **not** bumped, since nothing was observed. Measured in a WEAPONS critic review round 5: the **48 dead overrides** `#2` counted and `#3` confirmed unchanged (6 material instances × 8 parameters the master no longer declared) are now **0**. Every entry in `overrides.scalar` and `overrides.vector` on all six weapon MIs appears in the parent's `parameters[]`, and an independent check that does not go through the verb agrees: a byte scan of each `.uasset`'s name table against the master's 31 declared parameter names finds **zero** undeclared parameter names. So the game-side rot `#1`–`#3` were measuring got cleaned up. **What that means for this ticket:** the defect is about *presentation* — an orphaned override reading identically to a live one — and it was only ever observable because this project happened to hold 48 of them. With the rot gone, whether `get_material_instance_info` would **flag** an orphan is **untestable on this kit**, in either direction: a green result here now proves nothing, because there is nothing left to miss. This is the mirror of the caveat on `B-orphan-finder-vs-decompiler-disagree #13`, filed the same round — both tickets have run out of natural test material and both will keep returning uninformative clean results if re-measured on this project. **What verification now requires:** a **synthetic fixture** — author a master with a parameter, instance it, override that parameter, delete the parameter from the master, then read the instance back and check whether the dead override is marked. That is three or four calls and is the only thing that can close this. **Correcting nothing in `#1`–`#3`:** their measurements stand and were re-derived where cheap; this entry only records that the subject has been removed from under them, so a future round should not read "0 orphaned overrides" as evidence the verb improved. No plugin source was opened for this entry; the `overrides` / `parameters[]` comparison is read off `material.authoring.get_material_instance_info` responses and the name-table scan was run directly against the `.uasset` bytes.
