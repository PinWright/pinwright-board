---
id: F-substrate-mgir-sugar
title: "MGIR Substrate-material sugar, sanity warning, and wiki overlay"
status: DONE
severity: Low
category: feature
tags: [mgir, material, substrate, wiki, polish]
---

# MGIR Substrate-material sugar, sanity warning, and wiki overlay

**This is NOT a new IR.** Substrate materials already decompile correctly
through MGIR's existing reflection-driven path. The 23
`UMaterialExpressionSubstrate*` subclasses in UE 5.6 all derive from
`UMaterialExpression` and live in `UMaterial::Expressions` like every
other node, so `FExpressionInputIterator` walks their pins and
`AppendReflectedExpressionProperties` walks their UPROPERTYs without any
Substrate-specific code. Today they round-trip as plain
`call SubstrateSlabBSDF { DiffuseAlbedo: %albedo; F0: %f0; ... }` form,
and `MGIRDecompiler.cpp:727` already includes `FrontMaterial` in the
material-root-output list (verified). A parallel "SMGIR" namespace would
duplicate ~600–1000 LoC of emitter scaffolding for zero capability gain.

This ticket is presentation polish only, to let the plugin advertise
"Substrate material support" for Fab buyers on UE 5.4+ projects.

Three small additions:

1. **Syntactic-sugar layer (~80 LoC, optional).** Emit shorthand keywords
   when class name matches a Substrate slab/mixing/layering node:
   `slab`, `mix_h`, `mix_v`, `layer`, `weight`, `add`, `select`. Keep
   `call SubstrateSlabBSDF` as the canonical, round-trippable form — the
   sugar is recognised by the compiler but the decompiler can be put
   behind a flag if needed. Pure presentation, no semantic change.
2. **Sanity warning (~10 LoC).** When a decompiled material contains
   Substrate expressions but `r.Substrate=0` in project settings, attach
   a warning to the result payload. Do **not** hard-skip — Substrate
   expressions can legally exist as orphan data even with the feature
   disabled, and decompile should still surface them.
3. **Wiki overlay (~150 lines markdown).** Document the slab / mixing /
   layering vocabulary, the `FrontMaterial` and `SurfaceThickness` root
   outputs, and how the Substrate topology tree is built at
   shader-compile time from `FExpressionInput` connections (i.e. not a
   parallel authoring structure).

Effort: ~0.5 day total. Broad UE5 audience — UE 5.4 first shipped
Substrate as Experimental, UE 5.6 promoted it to Beta.

**Fix:** add sugar mapping table next to existing class-name dispatch in
the MGIR emitter; add `r.Substrate` CVar check in the decompile entry
point; add `docs/wiki/material.substrate.md` overlay.

## History
- `#1-initial-proposal` `OPEN` reporter — Filed after research audit. Verified MGIRDecompiler.cpp:727 includes FrontMaterial in root-output list. No existing substrate ticket in board. Substrate already decompiles correctly today; this ticket only adds shorthand emission, a project-settings sanity warning, and a wiki page so Fab buyers can find the feature.
- `#2-substrate-mgir-sugar` `IN-REVIEW` developer — Added the shared Substrate sugar helper for `slab`, `mix_h`, `mix_v`, `layer`, `weight`, `add`, and `select`; wired opt-in decompile sugar plus assigned-form parser lowering to canonical `call`; added the `r.Substrate=0` warning payload, MGIR/Substrate docs, and regression coverage for decompile sugar, compile sugar, and disabled-CVar warnings.
- `#3-verify-substrate-sugar` `DONE` tester — Verified: `material.compile_mgir` accepted `%slab = slab()` and created 1 expression at `/Game/App/UI/Test/M_McpVerifyTemp_F_substrate_mgir_sugar`; with `r.Substrate=0`, `material.decompile_mgir` using `emitSubstrateSugar:true` returned `slab()`, `output FrontMaterial`, and warning `Material contains Substrate expressions but r.Substrate=0`; temp asset was deleted via `asset.delete`.
