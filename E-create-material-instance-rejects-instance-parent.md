---
id: E-create-material-instance-rejects-instance-parent
title: "`material.authoring.create_material_instance` refuses a MaterialInstanceConstant as `parentMaterial` while `set_material_instance_parent` accepts one, so building an instance-of-an-instance takes a create-then-reparent two-step"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-instance, create_material_instance, set_material_instance_parent, parent-class, asymmetry, two-step, workaround]
encounters: 1
lastSeen: 2026-09-03T04:18:09Z
---

# One verb rejects the parent class the sibling verb accepts

`material.authoring.create_material_instance` requires `parentMaterial` to be a `UMaterial` master.
Reproduced:

```
material.authoring.create_material_instance
  {name: "MI_WPN_ParentProbe", path: "/Game/FPS/Weapons/Materials/Test",
   parentMaterial: "/Game/FPS/Weapons/Materials/MI_WPN_Polymer", save: false}

[UNSUPPORTED_ASSET_CLASS] parentMaterial is not a Material.
Received class: MaterialInstanceConstant. This verb parents to a UMaterial master only.
```

`material.authoring.set_material_instance_parent` accepts exactly that class without complaint.
Measured on the resulting tree — four instances built this way, each reparented onto a
`MaterialInstanceConstant`, read back by walking `UMaterialInstance::Parent`:

```
MI_VM_WPN_MetalAnodised   -> MI_WPN_MetalAnodised  -> M_WPN_Master
MI_VM_WPN_MetalPhosphate  -> MI_WPN_MetalPhosphate -> M_WPN_Master
MI_VM_WPN_OpticGlass      -> MI_WPN_OpticGlass     -> M_WPN_Master
MI_VM_WPN_Polymer         -> MI_WPN_Polymer        -> M_WPN_Master
```

So the two-level chain is fully supported by the plugin and by the engine; only the *creator* refuses
to name it. The engine's own `UMaterialInstance::Parent` is a `UMaterialInterface`, which is what
makes an instance-of-an-instance legal in the first place.

## Why this is not just a docs nit

The restriction is documented — `Docs/wiki-src/material.authoring.create_material_instance.md`
already says `parentMaterial` must be a master "even though the engine's own
`UMaterialInstance::Parent` is a `UMaterialInterface` and would allow an instance-of-an-instance". A
documented asymmetry is still an asymmetry: the caller must know to create against a master they do
not want and then reparent, and nothing in the error message says that. The error names the class it
rejected but not the verb that would accept it.

**The workaround costs correctness, not just a round-trip.** The create call has a `parameters` slot
for inline overrides applied at creation. Those overrides are authored against the *master's*
parameter set, then the asset is reparented onto an instance — so any override written at create time
is being written against the wrong parent, and whether it survives depends on `preserveOverrides` on
the reparent. In the measured case it did (all four report their static switch intact afterwards),
but that is a property of that call, not a guarantee the shape gives you.

## Cost

Hit while wiring a viewmodel-FOV material chain: four instances that each had to inherit a
per-slot-tuned parent's look. Parenting to the master instead would have meant hand-copying every
override and then keeping two sets in sync forever, so the two-step was the only correct route and
had to be discovered by trying the obvious thing and reading the failure.

## Fix

Accept `UMaterialInterface` for `parentMaterial` — the class the engine's own field takes — and let
the create path do in one call what create-then-reparent already does in two. If the restriction is
deliberate, the error message should name `set_material_instance_parent` as the route, the way the
`UMaterial`-only verbs already name `set_material_instance_base_property_overrides` when handed an
instance.

## Related

- `E-create-material-instance-duplicate-divergent-shape` — the same verb's other ergonomics; both of
  its complaints are now fixed (see its `#N` note), which leaves this as the remaining rough edge on
  the creator.

## History
- `#1-creator-rejects-instance-parent-sibling-accepts-it` `OPEN` reporter - `material.authoring.create_material_instance` returns `UNSUPPORTED_ASSET_CLASS` ("parentMaterial is not a Material. Received class: MaterialInstanceConstant. This verb parents to a UMaterial master only") for a `MaterialInstanceConstant` parent, while `material.authoring.set_material_instance_parent` accepts the same class. Both halves reproduced: the refusal directly, and the acceptance by walking `UMaterialInstance::Parent` on four instances built that way, each sitting `MI_VM_WPN_* -> MI_WPN_* -> M_WPN_Master`. The two-level chain is supported by the engine (`Parent` is a `UMaterialInterface`) and by the plugin's own reparent verb; only the creator refuses to name it. The restriction is documented in the wiki source but the error message is not - it names the class rejected, not the verb that would accept it. The workaround is not merely an extra round-trip: the create call's inline `parameters` overrides are authored against the master's parameter set and then the asset is reparented onto an instance, so override survival depends on `preserveOverrides` rather than on the shape being correct. Cost: four viewmodel instances that had to inherit a per-slot-tuned parent, where parenting to the master would have meant hand-copying every override and keeping two sets in sync permanently. Fix: accept `UMaterialInterface`, or name `set_material_instance_parent` in the error the way the `UMaterial`-only verbs already name `set_material_instance_base_property_overrides`.
