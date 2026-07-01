---
id: F-crir-hierarchy-mutation-write
title: "CRIR Phase B — rig hierarchy mutation on compile"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, hierarchy, roundtrip]
---

# CRIR Phase B — rig hierarchy mutation on compile

CRIR Phase A (shipped via `F-control-rig-ir-language`) emits the
`rig_hierarchy { ... }` block during decompile, but the compiler refuses to
mutate the hierarchy: any non-empty `rig_hierarchy` block in the input fails
with `CRIR_HIERARCHY_NOT_WRITABLE`. Round-trip is therefore restricted to rigs
whose hierarchy already exists (typically imported from a SkeletalMesh).

Phase B closes the loop: drive `URigHierarchyController::AddBone` /
`AddNull` / `AddControl` / `AddSocket` from compiled `bone` / `null` /
`control` / `socket` instructions inside `rig_hierarchy`. The mutation surface
is sized by Control settings and values:

- `URigHierarchyController::AddControl(InName, InParent, InSettings, InValue, InOffsetTransform, InShapeTransform, ...)` takes an `FRigControlSettings` (≈40 fields: control type enum, primary/secondary axis, animation/visibility/limits/shape flags, gizmo color, channel-disabled set, value-min/max/initial as `FRigControlValue`s, etc.) and an `FRigControlValue` (tagged-union over Bool/Float/Int/Vector2D/Vector/Rotator/Transform/EulerTransform).
- The other `Add*` calls are smaller (name + parent + transform + element-type metadata) but still need transform-vs-default canonicalization to keep round-trip text minimal.

The grammar already reserves `parent`, `location`, `rotation`, `scale`, and
`shape` attributes (see `crir-language-reference.md`). Phase B extends the
attribute schema to cover the full `FRigControlSettings` field set with
key=value pairs (mirror MGIR's struct-field expansion), and adds an
`FRigControlValue`-typed literal grammar for `value=`/`min=`/`max=`/`initial=`.

## Open design questions

- Whether to flatten `FRigControlSettings` fields into a flat attribute set
  (matches the existing parser shape) or nest them under a `settings { ... }`
  sub-block (matches AGIR's nested-block convention). Flat is simpler for
  Phase A's parser; nested may scale better as fields proliferate.
- How to encode `FRigControlValue`'s tagged-union over 8 underlying types in
  text without regressing round-trip determinism. Likely a typed prefix
  (e.g. `value=transform(...)`).
- Whether `bone`/`null`/`socket` get the same flat-attribute treatment or
  follow a narrower form (only `parent` + transform — they have far fewer
  metadata fields).

**Fix (wave 1 shipped):** Widened `CRIRCompiler::CompileRigHierarchyBlock` to
take `UControlRigBlueprint*` plus a warnings ref and to drive
`URigHierarchyController` (obtained via `Blueprint->GetHierarchyController()`)
for `bone`, `null`, and `socket` elements. Parent resolution prefers the
in-pass `AddedByName` cache and falls back to `URigHierarchy::Find` across
`{Bone, Null, Control, Socket}` element types. Transform attributes
(`location`/`rotation`/`scale`) decode via a new `TryParseVec3Tuple` helper
with `CRIR_HIERARCHY_BAD_TRANSFORM` on shape failure. The `control` element
continues to raise `CRIR_HIERARCHY_NOT_WRITABLE` and is carved out into a
follow-up ticket `F-crir-control-mutation-write` — the `FRigControlSettings`
(~40 fields) and `FRigControlValue` (8-variant tagged-union) surface roughly
doubles the test scope and warrants its own sprint. A new round-trip test
`FCRIRRoundTripHierarchyTest` covers bone/null/socket equality. Design
questions about flat-vs-nested attributes and `FRigControlValue` encoding
move with the `control` element to the follow-up ticket.

## Depends on

- `F-control-rig-ir-language` (Phase A) must have landed.

## History
- `#1-initial-spec` `OPEN` developer — Filed as the explicit Phase B carve-out
  during the Phase A sprint per the no-silent-defer rule. Phase A scope was
  capped at decompile-only hierarchy because the `FRigControlSettings` /
  `FRigControlValue` surface roughly doubles the test surface and offers no
  value until a user needs hierarchy authoring through the gateway. Defer until
  concrete demand surfaces.
- `#2-wave-1-bone-null-socket` `IN-REVIEW` developer — Wave 1 scope cut: shipped bone/null/socket mutation via URigHierarchyController; widened CompileRigHierarchyBlock to take UControlRigBlueprint*; control element continues to raise CRIR_HIERARCHY_NOT_WRITABLE pointing at F-crir-control-mutation-write. New round-trip test FCRIRRoundTripHierarchyTest covers bone+null+socket.
- `#3-verify-fix` `DONE` tester — Verified: created temp `/Game/App/UI/Test/CR_McpVerifyTemp_F_crir_hierarchy`, ran `controlrig.compile_crir` with `rig_hierarchy { bone "root" / null "ctrl_root" parent="root" location=(0,0,12) / socket "attach_point" parent="root" }` → `blocksCompiled=1`, no errors/warnings. Round-trip `controlrig.decompile_crir` returned `bone root`, `socket attach_point parent=root`, `null ctrl_root parent=root location=(0,0,12)` — all three element kinds mutated and parent + transform attributes preserved. Temp asset deleted.
