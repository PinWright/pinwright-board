---
id: B-convert-to-skeletal-mesh-leaves-sections-without-a-material-slot
title: "geometry.convert_to_skeletal_mesh writes a mesh with MORE render sections than material slots and a NULL slot 0, so part of the mesh renders as WorldGridMaterial forever - reported as success, and SetMaterial on the component cannot reach it"
status: OPEN
severity: High
category: bug
tags: [geometry, convert-to-skeletal-mesh, skeletal-mesh, material-slots, sections, boolean, viewmodel, silent-success]
encounters: 1
lastSeen: 2026-09-05T18:05:00Z
---

# A converted skeletal mesh gets 2 sections and 1 material slot, and the slot is null

## What the asset looks like

`skeleton.describe_mesh {skeletalMeshPath: "/Game/FPS/Player/SKM_FPSArms"}` on the asset
`geometry.convert_to_skeletal_mesh` produced:

```
materials:      [ { slot: "MaterialSlot", path: null } ]      <- ONE slot, and it is NULL
sectionsByLod:  [ 2 ]                                          <- TWO render sections
lods: 1, trianglesByLod: [38704], verticesByLod: [21144]
```

Two consequences, both silent:

1. **Slot 0 has no material at all.** Every section that resolves to index 0 renders the engine
   default (`WorldGridMaterial`).
2. **Section 1 has no slot to resolve to.** Index 1 is past the end of a one-entry array, so it also
   falls back to the default - and no component-level override can fix it, because
   `SetMaterial(0, ...)` only rewrites index 0.

The create path reported `materialSlots: 1, materialsPreserved: false` and the later overwrite
reported `materialSlots: 1, materialsPreserved: true`. Neither mentions the section count, and
nothing in the chain warns that sections outnumber slots.

## How it presents, which is why it cost two builds

The mesh renders as pale chalky grey with fine speckle - `WorldGridMaterial` on a skinned surface.
It looks exactly like a badly-tuned material, not a missing one, so the natural response is to keep
tuning the material. I did that twice across two builds:

- verified the instance resolved `SleeveTint` = (0.035, 0.038, 0.045) and `Roughness` 0.62 through
  `MaterialEditingLibrary` - correct, and irrelevant;
- retuned tiling 9 -> 2, `Specular` 0.32 -> 0.10, `Roughness` 0.62 -> 0.80 - no visible change;
- suspected the capture exposure and replaced an `AutoExposureBias` hack with a properly pinned
  `editor.screenshot {exposure: {mode: "fixed", ev100: 0}}` - correctly exposed frame, same pale
  arms;
- hid the third-person body to rule it out - frame identical.

**The measurement that broke it open:** assigning a completely different, dark, metallic material to
the component at runtime (`arms.set_material(0, MI_WPN_MetalPhosphate)`) changed **nothing** on
screen, while `arms.get_material(0)` cheerfully reported the new material. A component that reports
material X and renders the default material is the signature of a section index with no slot behind
it. `skeleton.describe_mesh` then showed `sectionsByLod: [2]` against one slot.

## Asked for

1. `geometry.convert_to_skeletal_mesh` must emit **at least as many material slots as the mesh has
   sections**, and must never write a null slot. Defaulting the extra slots to the source mesh's
   material, or to `WorldGridMaterial` explicitly, is fine - what is not fine is an out-of-range
   section index.
2. Failing that, **report it**: `sectionCount` alongside `materialSlots` in the response, and a
   warning when `sectionCount > materialSlots` or any slot is null. The caller cannot see this from
   any field the verb currently returns.
3. `skeleton.describe_mesh` already reports both numbers, which is how this was finally found - a
   line in the convert verb's page pointing at it would shorten the next person's search.

## Workaround

Add the missing slots by hand after the convert and re-verify with `skeleton.describe_mesh`:

```python
mats = list(sm.get_editor_property('materials'))
while len(mats) < section_count:
    m = unreal.SkeletalMaterial(); m.set_editor_property('material_interface', mi); mats.append(m)
for m in mats: m.set_editor_property('material_interface', mi)
sm.set_editor_property('materials', mats)
```

Verified on disk afterwards: `SKM_FPSArms.uasset` 5 130 802 B, `describe_mesh` reports two slots
both pointing at `MI_FPSArms`.

severity rationale: impact=ships a mesh that renders as the default material with no way to fix it
from the component API, while every verb in the chain reports success x reach=any
`convert_to_skeletal_mesh` output whose geometry produced more than one section, which a boolean
routinely does -> High

## History
- `#1-filed` `OPEN` reporter — Found on the FPS PLAYER stream after two builds of chasing it as a material-tuning problem. The mesh is the arms-only viewmodel cut from `SKM_Manny` with two `boolean_subtract` passes (see `F-isolate-skeletal-mesh-region-arms-only-viewmodel` #5); the boolean produced two sections and the convert wrote one null slot. Numbers above are from `skeleton.describe_mesh` before and after the manual fix, and the decisive runtime probe is the stock-material swap that changed nothing on screen.
- `#2-correction-the-defect-is-real-but-it-was-not-the-visual` `OPEN` reporter — **Correcting `#1`'s "How it presents" section.** The asset defect is exactly as reported and reproduces: `convert_to_skeletal_mesh` wrote one **null** material slot against `sectionsByLod: [2]`, and the ask stands unchanged. But it was **not** what made the arms render pale. After filling the null slot and adding the second (both `MI_FPSArms`, verified on disk), a fresh capture showed the surface unchanged, and assigning a completely different dark material to **both** elements changed nothing either. The actual cause was a broken shader on `M_FPSArms` — a `StaticSwitchParameter` whose `A` input was never connected, because `connect_nodes` accepted the pin name `True` that `get_material_node_details` itself reports; filed as `B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing`. A material that fails to compile renders as the engine Default Material, which on a skinned mesh is the same pale grey I attributed here. Two independent defects on one mesh, and the loud one masked the quiet one. Leaving this ticket open on its own merits: a null slot, and a section with no slot behind it, are still wrong, still reported as success, and would still strand a section on the default material once the shader is fixed.
