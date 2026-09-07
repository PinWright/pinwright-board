---
id: B-convert-to-skeletal-mesh-leaves-sections-without-a-material-slot
title: "geometry.convert_to_skeletal_mesh writes a mesh with MORE render sections than material slots and a NULL slot 0, so part of the mesh renders as WorldGridMaterial forever - reported as success, and SetMaterial on the component cannot reach it"
status: IN-REVIEW
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

## Fix

The shared skeletal asset-create seam now derives the required material-slot count from dynamic-mesh
material IDs and supplies deterministic placeholder slots before the engine build, so a sparse
section index cannot be clamped before repair. It then measures imported section material indices
and, when render data is available, validates render-section indices after the build. It fills null
entries and pads deterministic slots through the highest referenced index using an existing source
material or the engine surface default. The convert handler applies the same repair to overwrite
writes and returns the final `materialSlots` count. Its handler-harness regression fixture uses
sparse material IDs, saves the result, resets package loaders, then invokes the production
`asset.reload` handler and reacquires the asset before inspecting both imported and render sections
for in-range, non-null materials. The skeletal handler now uses the
central `ErrorCodes::ERR_*` registry for all emitted error codes. Scoped verification reports build
`Result: Succeeded` from `X:\src\unreal\unreal-fpv-dev\Saved\Logs\ubt-wave10-sk-iter1.log` and
`COMPLETED_CLEAN 4/4` from
`X:\src\unreal\unreal-fpv-dev\Saved\Logs\pw_sk_iter1.log`; no MCP or Git action was performed.

## History
- `#1-filed` `OPEN` reporter — Found on the FPS PLAYER stream after two builds of chasing it as a material-tuning problem. The mesh is the arms-only viewmodel cut from `SKM_Manny` with two `boolean_subtract` passes (see `F-isolate-skeletal-mesh-region-arms-only-viewmodel` #5); the boolean produced two sections and the convert wrote one null slot. Numbers above are from `skeleton.describe_mesh` before and after the manual fix, and the decisive runtime probe is the stock-material swap that changed nothing on screen.
- `#2-correction-the-defect-is-real-but-it-was-not-the-visual` `OPEN` reporter — **Correcting `#1`'s "How it presents" section.** The asset defect is exactly as reported and reproduces: `convert_to_skeletal_mesh` wrote one **null** material slot against `sectionsByLod: [2]`, and the ask stands unchanged. But it was **not** what made the arms render pale. After filling the null slot and adding the second (both `MI_FPSArms`, verified on disk), a fresh capture showed the surface unchanged, and assigning a completely different dark material to **both** elements changed nothing either. The actual cause was a broken shader on `M_FPSArms` — a `StaticSwitchParameter` whose `A` input was never connected, because `connect_nodes` accepted the pin name `True` that `get_material_node_details` itself reports; filed as `B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing`. A material that fails to compile renders as the engine Default Material, which on a skinned mesh is the same pale grey I attributed here. Two independent defects on one mesh, and the loud one masked the quiet one. Leaving this ticket open on its own merits: a null slot, and a section with no slot behind it, are still wrong, still reported as success, and would still strand a section on the default material once the shader is fixed.
- `#3-material-slot-coverage` `IN-REVIEW` developer — Changed the shared skeletal create seam and convert overwrite path to fill null slots and preserve sparse section material indices through the required slot count; retained the final `materialSlots` response field, added imported/render-section coverage to the handler-harness regression test, and documented the material-slot invariant. Live verification is intentionally pending; no build, editor run, or automation execution was performed.
- `#4-verifier-follow-up` `IN-REVIEW` developer — Corrected the handler-harness control-flow/ownership defects found in review: the fixture is now held by `TStrongObjectPtr`, cleanup destroys the actor before resetting the fixture and deleting both assets, and the test requests/checks durable save, resets loaders, invokes production `asset.reload`, reacquires the mesh, and asserts imported/render coverage on the reloaded mesh. Converted all skeletal-handler error-code call sites to the existing central registry. No Unreal build, editor run, MCP call, automation execution, or commit was performed.
- `#5-param-contract` `IN-REVIEW` developer — Corrected both geometry asset-load registrations so `lodType` is declared as the string enum their handlers read; added the exact registration-shape ratchet exception and docs coverage for `MaxAvailable`, `HiResSourceModel`, `SourceModel`, and `RenderData`. This unblocks the handler-harness request at registration time; no Unreal build, editor run, MCP call, automation execution, or commit was performed.
- `#6-ratchet-correlation` `IN-REVIEW` developer — Re-read the parameter ratchet and preserved the `lodType` exclusion only in the name-based `IsDiscreteReaderName` shape heuristic. The final declaration loop now runs through the actual `GetInt`/`GetIntOr`/`RequireInt` scan and re-admits exact `lodType`, so a future integer reader cannot bypass the integer declaration assertion. No Unreal build, editor run, MCP call, automation execution, or commit was performed.
- `#7-prebuild-material-coverage` `IN-REVIEW` developer — The C4 log showed the sparse fixture reached the skeletal builder as sections with material IDs 0 and 2, but the create seam still supplied only one implicit material slot when no explicit slot list was provided. That allowed the engine lifecycle to clamp the sparse section before post-build padding. The seam now preallocates deterministic slots through the dynamic mesh's highest material ID before its single build; the existing post-build null-slot/index-range repair remains in place. Source-only correction: no build, editor run, MCP call, automation execution, or commit was performed.
- `#8-scoped-verification` `IN-REVIEW` developer — Scoped verification reports build `Result: Succeeded` from `X:\src\unreal\unreal-fpv-dev\Saved\Logs\ubt-wave10-sk-iter1.log` and checker group `COMPLETED_CLEAN 4/4` from `X:\src\unreal\unreal-fpv-dev\Saved\Logs\pw_sk_iter1.log`; all four `PinWright.geometry.convert_to_skeletal_mesh` tests, including `MaterialSlotsCoverSections`, completed successfully. The ticket remains `IN-REVIEW`; the developer does not set `DONE`.
