---
id: E-material-set-vector-parameter-no-value-echo
title: "material.authoring.set_vector_parameter_value returns no value echo while its scalar sibling returns one from the same handler file, so the one setter whose value cannot be checked by grep is also the one that confirms nothing"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-authoring, set_vector_parameter_value, set_scalar_parameter_value, echo, readback, verification, asymmetry]
encounters: 1
lastSeen: 2026-09-06T00:31:00+03:00
---

# The scalar setter echoes what it wrote; the vector setter does not

Two calls, same asset, same session, same handler file:

```
material.authoring.set_scalar_parameter_value {parameterName:"AmbientBoost", value:1.55}
-> {... "parameterName":"AmbientBoost", "value":1.55}

material.authoring.set_vector_parameter_value {parameterName:"DustTint", value:{r:0.96,g:0.98,b:1,a:1}}
-> {... "parameterName":"DustTint"}                      <- no value, no previousValue
```

The scalar response carries `value`, so a caller can confirm the write — including
any coercion — from the result. The vector response carries only the parameter
name, which proves the handler resolved a parameter, not what it stored in it.
`set_texture_parameter_value` should be checked for the same shape.

## Why the asymmetry costs more here than on the scalar path

The vector setter is the one whose value is hardest to verify out of band.

- A scalar can be confirmed by packing one float and searching the `.uasset`:
  `struct.pack('<f', 1.55)` found exactly one hit, and the old `0.70` zero hits.
- A `FLinearColor` needs four packed floats at contiguous offsets, and each
  component is individually a common constant — `1.0` alone matched 1359 times in
  an 13 KB material instance. Confirming `DustTint` required scanning for
  `0.96/0.98/1.0/1.0` *and* checking the four offsets were adjacent
  (13065/13069/13073/13077) before the match could be trusted.

So the setter that echoes nothing is the setter whose write is least checkable by
the fallback the project's `CLAUDE.md` mandates ("verify a write against disk, not
against the object you just wrote"). The remaining confirmation route is a second
`material.authoring.get_material_instance_info`, which is an in-memory read and by
that same rule is not proof of a write.

## What it should do

Echo `value` (and, matching `niagara.set_module_input`'s `replacedOverride`, the
`previousValue` when the parameter was already overridden) in the canonical
`{r,g,b,a}` shape the verb accepts. The scalar handler already does exactly this,
so the shape is settled and the change is local to the vector (and texture)
handler.

## Distinct from

- `E-niagara-set-property-no-value-echo` (OPEN) — same defect shape, other
  namespace: `niagara.set_property` echoes neither value nor previous value while
  `niagara.set_module_input` / `niagara.set_static_switch` echo both. That ticket
  is Niagara renderer/emitter properties; this one is the material instance
  parameter setters, where the *scalar* sibling in the same handler already
  echoes, so the inconsistency is internal to one verb family.
- `B-material-param-setters-wrong-class-error` (OPEN) — the same three setters,
  but about the error code returned for a wrong-class asset, not the success
  payload.
- `E-material-instance-info-orphaned-overrides` (OPEN) — about the *reader*
  (`get_material_instance_info`) conflating dead and live overrides; this is about
  the *writer* returning nothing to read back.

## Evidence

- Session: VFX build 08 water column fix on `EAContentExamples58` (UE 5.8),
  `/Game/FPS/VFX/Materials/MI_FPS_Water_Crown`. Three writes in one batch —
  `AmbientBoost 0.70 -> 1.55` and `EdgeSharpness 1.20 -> 0.80` (both echoed
  `value`), `DustTint (0.72,0.84,0.94) -> (0.96,0.98,1.00)` (no echo).
- Disk confirmation that the vector write did land, obtained the hard way:
  little-endian f32 scan of `MI_FPS_Water_Crown.uasset` found `0.96/0.98/1.0/1.0`
  at contiguous offsets 13065-13077 and zero occurrences of the three old
  components, with the file's mtime moved and its size 11036 -> 13590 bytes.

## History
- `#1-initial` `OPEN` reporter — First encounter, VFX build 08 water work on `EAContentExamples58` (UE 5.8). `material.authoring.set_vector_parameter_value` on `MI_FPS_Water_Crown.DustTint` returned `success` with `parameterName` and no `value`, in the same three-call batch where `set_scalar_parameter_value` echoed `value:1.55` and `value:0.8` for `AmbientBoost` and `EdgeSharpness`. No tool error and no retry: the cost was that the one write needing a four-float contiguous-offset byte scan to confirm was also the one the response could not confirm, while the two writes the response *did* confirm were the two a single `struct.pack('<f', x)` search settles anyway. Dedup: ripgrep across the board found `E-niagara-set-property-no-value-echo` (same shape, Niagara namespace), `B-material-param-setters-wrong-class-error` (same three setters, error path not success payload) and `E-material-instance-info-orphaned-overrides` (the reader, not the writer); nothing covering the material setters' success echo. Suggested fix: echo `value` plus `previousValue` from the vector handler, matching the scalar handler in the same file; check `set_texture_parameter_value` for the same omission.
