---
id: E-create-material-instance-rejects-mi-parent
title: "create_material_instance refuses a MaterialInstanceConstant as parentMaterial, but set_material_instance_parent accepts one — so MI-of-MI is reachable only as a two-call workaround"
status: OPEN
severity: Low
category: enhancement
tags: [material, material-authoring, create-material-instance, material-instance-parent, mi-chain, multi-agent]
encounters: 1
lastSeen: 2026-09-03T04:14:00Z
---

# `create_material_instance` will not parent to an MI that `set_material_instance_parent` will

## Repro

UE 5.8, EAContentExamples58, PinWright on 27145.

```
material.authoring.create_material_instance {
  name: "MI_VM_WPN_MetalAnodised", path: "/Game/FPS/Player",
  parentMaterial: "/Game/FPS/Weapons/Materials/MI_WPN_MetalAnodised",
  parameters: { staticSwitch: { EnableViewmodelFOV: true } } }

-> [UNSUPPORTED_ASSET_CLASS] parentMaterial is not a Material.
   Received class: MaterialInstanceConstant. This verb parents to a UMaterial master only.
```

The engine has no such restriction — `UMaterialInstance::Parent` is a `UMaterialInterface`, and MI
chains are ordinary UE practice. And the plugin's own sibling verb allows exactly what this one
refuses:

```
material.authoring.create_material_instance   { parentMaterial: <the MASTER> }        -> ok
material.authoring.set_material_instance_parent { parentMaterial: <an MI> }           -> ok
  -> {"parent": "/Game/FPS/Weapons/Materials/MI_WPN_MetalAnodised.MI_WPN_MetalAnodised",
      "preserveOverrides": true}
```

So the capability is present; only the creator's validation disagrees with it.

## Why the two-call workaround is not free

Four instances built this way (`MI_VM_WPN_MetalAnodised`, `_Polymer`, `_MetalPhosphate`,
`_OpticGlass`) each spend a create + a reparent + a save, and each **transiently exists parented to
the wrong material**. That window matters in a shared editor: another stream reading the asset
between the two calls sees an instance whose parent, and therefore whose whole parameter set, is
not what it will be a second later. It also inverts the ordering you want for
`parameters` — the inline overrides are applied against the master, then the parent changes under
them; `preserveOverrides: true` held for a static switch in this case, but that is the creator's
promise about a parent it was not created against.

## Context where it came up

Building first-person viewmodel variants of another stream's weapon materials. Parenting to their
`MI_WPN_*` rather than to `M_WPN_Master` is the whole point: the viewmodel copy then inherits their
per-slot tuning automatically and adds exactly one override (`EnableViewmodelFOV = true`), instead
of my hand-copying every parameter and owning a sync problem between two streams forever. That is
the normal reason to want MI-of-MI, and it is a two-agent workflow, not an exotic one.

## Asked for

Accept any `UMaterialInterface` as `parentMaterial` and apply `parameters` against the real parent,
so the instance is correct from its first save. If the restriction is deliberate, the error should
say so and point at the two-call route rather than asserting "parents to a UMaterial master only" as
though the alternative did not exist.

severity rationale: impact=cosmetic, a documented workaround exists and works x reach=any project
building instance chains, which is standard material practice -> Low

## History
- `#1-filed` `OPEN` reporter — Found on the FPS PLAYER stream building four viewmodel-FOV variants of the WEAPONS stream's `MI_WPN_*` materials. Workaround (create against `M_WPN_Master`, then `set_material_instance_parent` to the MI) produced correct assets: all four verified on disk with the parent and `EnableViewmodelFOV=True` read back through `MaterialEditingLibrary.get_material_instance_static_switch_parameter_value`, and the WEAPONS-owned originals confirmed untouched by mtime.
