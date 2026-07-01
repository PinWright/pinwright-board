---
id: F-crir-control-mutation-write
title: "CRIR Phase B — control-element mutation on compile"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, hierarchy, roundtrip]
---

# CRIR Phase B — control-element mutation on compile

Carved out from `F-crir-hierarchy-mutation-write` during the wave-1 sprint that
shipped `bone` / `null` / `socket` mutation. The remaining scope is to drive
`URigHierarchyController::AddControl` from compiled `control` instructions
inside `rig_hierarchy`. Until this lands, a non-empty `control` element in
CRIR input continues to error with `CRIR_HIERARCHY_NOT_WRITABLE`.

The mutation surface is sized by Control settings and values:

- `URigHierarchyController::AddControl(InName, InParent, InSettings, InValue, InOffsetTransform, InShapeTransform, ...)` takes an `FRigControlSettings` (~20 UPROPERTY fields: control type enum, primary axis, animation/visibility/draw-limits/shape flags, gizmo color, filtered-channels list, value-min/max as `FRigControlValue`s, etc.) and an `FRigControlValue` (templated `Set<T>` over storage types `bool`/`float`/`int32`/`FVector3f`/`FTransform_Float`/`FTransformNoScale_Float`/`FEulerTransform_Float`).
- The smaller `Add*` calls (`AddBone`/`AddNull`/`AddSocket`) are already wired by wave 1 using a flat `key=value` attribute set for `parent`/`location`/`rotation`/`scale`. Wave 1 implicitly answered the third question from the parent ticket (narrower form for non-control kinds) by using that same flat shape.

## Resolved architecture — Hybrid + typed-prefix values

Three encodings were researched (pure flat, pure nested, hybrid). Hybrid
won on MCP usability, round-trip parity, and parser cost amortization. The
key insight: in practice the trivial-control case dominates real Control Rig
assets, and pure flat collapses to hybrid via elision while pure nested
breaks wave-1's one-line element convention even for trivial controls.

**Top-level (hot path).** Four reserved flat attributes on the `control`
element line, in canonical order:
1. `parent="<name>"` (wave-1 reuse)
2. `type=<variant>` — value-type discriminator from `ERigControlType`
   (`bool`, `float`, `int`, `vector2d`, `position`, `scale`, `scale_float`,
   `rotator`, `transform`, `transform_no_scale`, `euler_transform`).
   Required because `value=` is otherwise ambiguous.
3. `value=<typed-prefix-literal>` — the current/initial value.
4. `shape=<typed-prefix-literal>` — gizmo offset transform (only emitted
   when non-default).

Plus the wave-1 element-line transform attributes `location` / `rotation` /
`scale` continue to mean the control's offset transform on the rig, matching
bone/null/socket convention exactly.

**Sub-block (long-tail).** Trailing `{ ... }` opens only when at least one
field of `FRigControlSettings` (or limit/min/max/animation customization)
is non-default. Inside, lines are `key=value` pairs alphabetically sorted by
the emitter. Defaulted fields are elided entirely; an empty sub-block is
forbidden — the braces are omitted. Sub-block keys (~20 total) cover:
`primary_axis` (no SecondaryAxis on FRigControlSettings),
`animation_type`, `display_name`, `control_enum`, `draw_limits`,
`limits=[(min=<bool>,max=<bool>),…]` — positional `TArray<FRigControlLimitEnabled>`,
one entry per channel of the resolved `ControlType` (size set by
`SetupLimitArrayForType`), each entry an explicit `{bMinimum, bMaximum}` pair
rather than a per-axis-name struct; `min`/`max` (typed-prefix literals matching
`type`), `filtered_channels=[…]`, `shape_name`, `shape_color`,
`shape_visible`, `shape_visibility`, `group_with_parent_control`,
`restrict_space_switching`, `is_transient_control`, `driven_controls`,
`preferred_rotation_order`, `use_preferred_rotation_order`. `ShapeTransform`
is **not** a sub-block key — it is `UPROPERTY(Transient)` on `FRigControlSettings`
and only flows out via the top-level `shape=` attribute (which maps to
`AddControl`'s `InShapeTransform` parameter).

**`FRigControlValue` typed-function-prefix grammar.** The bareword before
`(` is the variant tag; the parser dispatches on it to the matching
`FRigControlValue::Make<T>(value)` instantiation (the value type is
templated `Set<T>` over a 32-float storage blob, NOT eight typed-setter
overloads). 11 spellings, 1:1 with `ERigControlType`:

```
value=bool(true)
value=float(1.5)
value=int(3)
value=vector2d(1, 2)
value=position(1, 2, 3)
value=scale(1, 1, 1)
value=scale_float(0.5)
value=rotator(p=0, y=90, r=0)
value=transform(loc=(0,0,5), rot=(0,90,0), scale=(1,1,1))
value=transform_no_scale(loc=(0,0,0), rot=(0,0,0))
value=euler_transform(loc=(...), rot=(...), scale=(...))
```

Several share underlying storage: `vector2d`, `position`, `scale`, and
`rotator` all store as `FVector3f` (rotator packs pitch/yaw/roll into X/Y/Z;
vector2d zeros Z); `scale_float` shares storage with `float`;
`transform`/`transform_no_scale`/`euler_transform` use their respective
`*_Float` storage structs nested in `FRigControlValue`. The prefix carries
logical identity and the compiler dispatches on `ERigControlType` + the
matching underlying storage type to construct the `FRigControlValue`.

`min` / `max` / `initial` each carry their own prefix; the compiler validates
that all four agree with the element's `type=` and emits
`CRIR_CONTROL_VALUE_TYPE_MISMATCH` on disagreement — the most useful error
the parser can deliver. Positional args for fixed-shape low-arity types
(`vector(1,2,3)`); keyword args for compound types where order is a footgun
(`rotator(p=,y=,r=)`, `transform(loc=,rot=,scale=)`). Mirrors BPIR's
struct-literal grammar.

**Round-trip determinism.** Emitter rules: (a) top-level attributes always
in the fixed order above; (b) sub-block keys alphabetically sorted; (c)
field equal to `FRigControlSettings()` default of the same `ControlType`
elided; (d) empty sub-block omitted entirely; (e) float formatting via the
wave-1 canonical formatter. Two semantically identical controls produce
byte-identical text.

**Engine-version drift.** New `FRigControlSettings` UPROPERTYs in future UE
versions become new sub-block keys with no naming collision. Default-elision
keeps existing assets producing identical text. Unknown sub-block keys raise
a soft warning rather than a hard error (matches MGIR property-set drift
policy).

**Worked example — trivial:**
```crir
control "Strength" parent="Root" type=float value=float(0.5)
```

**Worked example — non-trivial:**
```crir
control "IK_Hand_L" parent="Hand_L_Offset"
    type=transform
    value=transform(loc=(0,0,5), rot=(0,0,0), scale=(1,1,1))
    shape=transform(loc=(0,0,0), rot=(0,0,0), scale=(2,2,2))
{
    animation_type=animation_control
    primary_axis=y
    limits=(ty=true)
    min=transform(loc=(0,-10,0))
    max=transform(loc=(0,10,0))
    filtered_channels=[tz, rz]
    shape_color=(0.2, 0.8, 0.2, 1.0)
    shape_name=Box_Solid
}
```

## Fix

Extend `CRIRCompiler::CompileRigHierarchyBlock` to dispatch
`ECRIRElementKind::Control` to `URigHierarchyController::AddControl`. Add
`FormatControlValue`, `FormatControlSettingsSubBlock`, and
`EmitControlElement` helpers to `CRIRTextEmitter` (handling the ~20
`FRigControlSettings` UPROPERTY fields — not 40; the higher count came from
counting `FRigControlValueStorage`'s 32 internal floats plus
transient/deprecated fields). Add an `FCRIRControlValueParser` helper that
dispatches on the 11 bareword prefixes (1:1 with `ERigControlType`;
scoped helper, ~200 LOC; reuses `FIrTextUtils::FindMatchingChar` to scope
the `(...)` payload). The value type is the templated
`FRigControlValue::Make<T>(value)` API operating on a 32-float storage blob
— **not** 8 typed setter overloads. Sub-block emits ~20 alphabetically
sorted, default-elided settings keys; `SecondaryAxis` is omitted (the
field does not exist on `FRigControlSettings` — only `PrimaryAxis`), and
`bDrawLimits` is included. `bIsCurve` (transient) and
`PreviouslyDrivenControls` (no UPROPERTY) are excluded along with the
deprecated `bAnimatable_DEPRECATED` / `bShapeEnabled_DEPRECATED`.
`ShapeTransform` is a transient field on Settings — emitted ONLY via the
top-level `shape=` attribute which maps to `AddControl`'s
`InShapeTransform` parameter (NOT as a sub-block key). The parser's N-deep
brace stack from `F-crir-collapse-and-functionref` is reused: element
trailing `{` pushes a new `FParseFrame` whose `ElementAttributes` aliases
the last element's `Attributes` map. Drop the
`CRIR_HIERARCHY_NOT_WRITABLE` error path entirely once control elements
round-trip. Add error codes: `CRIR_CONTROL_BAD_TYPE`,
`CRIR_CONTROL_BAD_VALUE`, `CRIR_CONTROL_VALUE_TYPE_MISMATCH` (for
`type=` / value-prefix disagreement), and `CRIR_CONTROL_BAD_SUBBLOCK_KEY`
(soft warning for unknown sub-block keys). Extend the round-trip test with
control fixtures covering all 11 `ERigControlType` variants and at least
one maximal-settings control.

## Depends on

- `F-crir-hierarchy-mutation-write` (wave 1: bone/null/socket).
- `F-crir-collapse-and-functionref` shares the trailing-`{` element
  sub-block parser primitive; whichever ticket lands first builds it.

## History
- `#1-initial-spec` `OPEN` developer — Carved out from F-crir-hierarchy-mutation-write during wave 1 sprint. The FRigControlSettings ~40-field surface plus the FRigControlValue 8-variant tagged-union roughly doubles the test surface and warrants its own sprint. The two remaining design questions in the parent ticket (flat vs nested attribute encoding, FRigControlValue prefix encoding) all concern control-element encoding and move here.
- `#2-architecture-chosen` `OPEN` developer — Architecture resolved via three parallel research subagents (flat / nested / hybrid). Hybrid + typed-function-prefix selected: top-level flat keeps wave-1 one-line element shape for trivial controls, sub-block only opens when settings diverge from defaults, typed-prefix values give the parser the tagged-union info needed to deliver precise errors. Round-trip determinism + engine-version drift + MCP/LLM write-path ergonomics all addressed. Ticket is now sprint-ready.
- `#3-corrections-from-impl-analysis` `IN-REVIEW` developer — Implementation pass corrected ticket inaccuracies against UE 5.6 surface: dropped SecondaryAxis (doesn't exist), added bDrawLimits, removed ShapeTransform from sub-block (transient field; flows only via top-level shape= → AddControl InShapeTransform param), corrected ERigControlType list to 11 variants (was 8 — added position/scale/scale_float/transform_no_scale), corrected FRigControlValue API to templated Set<T> + Make<T> (not 8 typed setters), excluded bIsCurve and PreviouslyDrivenControls (transient/no UPROPERTY), noted sub-block key count is ~20 not 40. Fix paragraph rewritten accordingly; worked examples unchanged in shape. Parser brace-stack from ticket 2 reused via FParseFrame::ElementAttributes slot.
- `#4-control-element-mutation-shipped` `IN-REVIEW` developer — Implemented control element compile path in CRIRCompiler.cpp (dropped CRIR_HIERARCHY_NOT_WRITABLE early-return). New FCRIRControlValueParser helper (CRIRControlValueParser.{h,cpp}) parses the 11 typed value prefixes via FIrTextUtils::FindMatchingChar scoping. New emitter helpers EmitControlElement, FormatControlValue, FormatControlSettingsSubBlock in CRIRTextEmitter. Parser extended to recognize element-trailing-{} via the ticket-2 N-deep brace stack (FParseFrame::ElementAttributes path). Sub-block emits ~20 settings keys alphabetically sorted, default-elided. New error codes: CRIR_CONTROL_VALUE_TYPE_MISMATCH, CRIR_CONTROL_BAD_TYPE, CRIR_CONTROL_BAD_VALUE, CRIR_CONTROL_BAD_SUBBLOCK_KEY (soft warning). Tests in TestCRIRControlMutation.cpp: 11 per-variant round-trips, maximal-settings, byte-equal full round-trip, type-mismatch error path.
- `#5-verify-fix` `DONE` tester — Verified: created temp `/Game/App/UI/Test/CR_McpVerifyTemp_F_crir_control`, ran `controlrig.compile_crir` with `rig_hierarchy` containing all 11 ERigControlType variants (bool/float/int/vector2d/position/scale/scale_float/rotator/transform/transform_no_scale/euler_transform). Compile returned `blocksCompiled=1, warnings=[]` and `controlrig.decompile_crir` round-tripped all 11 controls with typed `value=` literals plus default-elided sub-blocks (only `limits=` arrays sized per channel). Negative path: `type=float value=bool(true)` returned `CRIR_CONTROL_VALUE_TYPE_MISMATCH "value prefix 'bool' does not match type=float"`. Note: `nodesCreated` reported 0 despite 11 elements created — minor counter bug, doesn't block the feature. Temp asset deleted.
