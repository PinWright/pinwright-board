---
id: B-niagara-module-input-link-particle-attribute
title: "niagara.set_module_input cannot link a module input to a particle attribute — {link:\"Particles.NormalizedAge\"} is refused PARAMETER_NOT_FOUND even on a top-level float input whose type is known"
status: OPEN
severity: High
category: bug
tags: [niagara, set-module-input, link, particle-attribute, normalized-age, type-inference, parameter-not-found]
encounters: 1
lastSeen: 2026-09-02T22:45:00+05:00
---

# `niagara.set_module_input` refuses `{link: "Particles.<attr>"}`, so no module input can be driven by a particle attribute

`niagara.set_module_input`'s documented linked path (`value: { link: "User.X" }`)
resolves the linked parameter's type from a parameter store. `Particles.*`
attributes do not live in a store, so the resolver finds nothing and the call is
refused:

```
niagara.set_module_input {
  assetPath: "/Game/FPS/VFX/Emitters/E_ImpactDirt_Spray",
  entryId:   "6BAEC06146F6D69BA1DAFE9E57E8108D",   // ScaleSpriteSize
  inputName: "Uniform Curve Index",                 // declared NiagaraFloat
  value:     { link: "Particles.NormalizedAge" } }
-> [PARAMETER_NOT_FOUND] Cannot infer a type for linked parameter
   'Particles.NormalizedAge' (module input type unknown).
```

The parenthetical is the tell: the fallback is the **module input's declared
type**, and the handler does not have it — so even an input Niagara itself
declares as `NiagaraFloat` cannot be linked. The refusal is not specific to a
nested/compound input name; the call above names a plain top-level input on a
stock module.

## Why it matters

`Particles.NormalizedAge`, `Particles.Age`, `Particles.Velocity` and
`Particles.Position` are how Niagara content parameterises anything *over life*.
With them unreachable, the whole class of over-life authoring is closed through
the MCP:

- size-over-life (`ScaleSpriteSize` / `ScaleMeshSize` `Uniform Scale Factor`)
- alpha/colour-over-life through a `Lerp_Float` / `Lerp_Vector` alpha
- speed- or age-driven forces, drag ramps, kill conditions

The only remaining route to over-life behaviour is a dynamic input that reads
`Particles.NormalizedAge` **internally** — `/Niagara/DynamicInputs/Helpers/RampInOut`
or `/Niagara/DynamicInputs/ValueFromCurve/FloatFromCurve`. Both are worse:
`RampInOut`'s shape is governed by static switches that cannot be set at all on a
nested node (`F-niagara-dynamic-input-nested-inputs`), and `FloatFromCurve`'s curve
cannot be edited (`B-niagara-set-curve-keys-unreachable-module-input-di`). So the
workaround chain is itself blocked twice over.

## Repro

1. `asset.duplicate` `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst`
   to a scratch path.
2. `niagara.add_module` `/Niagara/Modules/Update/Size/ScaleSpriteSize`,
   `scriptUsage: "ParticleUpdateScript"` — keep the returned `nodeId`.
3. `niagara.set_module_input` on that `nodeId`, `inputName: "Uniform Curve Index"`
   (or any declared-float input), `value: { link: "Particles.NormalizedAge" }`.
4. **FAIL** `[PARAMETER_NOT_FOUND] Cannot infer a type for linked parameter
   'Particles.NormalizedAge' (module input type unknown).`
5. Same call with a literal (`value: 0.5`) succeeds and echoes `value:"0.5"`, so
   the module target and input name are both correct — only the link path fails.

## Expected

A link to a well-known `Particles.*` / `Engine.*` attribute resolves, either by

- consulting the emitter's compiled attribute set / `FNiagaraConstants` for the
  attribute's type, or
- falling back to the **module input's declared type** via
  `FNiagaraStackGraphUtilities::GetStackFunctionInputs` (the same discovery
  `F-niagara-dynamic-input-nested-inputs` names), or
- accepting an explicit `linkType` on the request so the caller can supply the
  type the handler cannot infer — the cheapest fix, and enough to unblock every
  case above.

Whatever the mechanism, an attribute Niagara writes on every particle should not
be less reachable than a user parameter.

## Distinct from related tickets

- `B-niagara-literal-over-linked-override-pin` and
  `B-niagara-link-modes-destroy-override-silently` are about what happens to an
  **existing** override when a link is written. This ticket is that the link is
  never written at all.
- `F-niagara-link-module-input-to-parameter` delivered the `User.*` link path;
  `Particles.*` was never covered.
- `B-niagara-module-input-stack-infer` is a stack-resolution defect
  (`INVALID_STACK`), a different failure on a different argument.

severity rationale: impact=hard blocker — over-life parameterisation is
unreachable and both documented workarounds are themselves blocked x reach=every
Niagara effect that changes with age, which in practice is every effect -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the FPS impact VFX systems under `/Game/FPS/VFX/` (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-09-02, UE 5.8, EAContentExamples58 checkout, live editor on port 27145. Exact failing call and error text recorded above; verified against a fresh `SimpleSpriteBurst` duplicate. Both a **compound** input name (`"Uniform Scale Factor.Alpha"`, the Alpha of an assigned `Lerp_Float`) and a **top-level** input name (`"Uniform Curve Index"`, declared `NiagaraFloat` on the stock `ScaleSpriteSize`) return the identical message, so this is not the nested-input limitation — the linked path has no type source for `Particles.*` in either position. Literal writes to the same inputs succeed and echo their pin defaults, and the writes were independently confirmed on the graph (`niagara.inspect {includeGraphs:true}` shows the override input nodes carrying `1.0` / `2.1`), so the target resolution is sound. Not source-confirmed: no `TryGetLinkedParameterRequest` read was made in this session, the diagnosis rests on the error text's own parenthetical.
