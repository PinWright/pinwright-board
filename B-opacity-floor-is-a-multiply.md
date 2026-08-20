---
id: B-opacity-floor-is-a-multiply
title: "OpacityFloor is a multiply, not a floor — the parameter caps opacity at its own value instead of guaranteeing it, so MI_PwModelExample_Glass tops out at 0.35 and the name promises the opposite"
status: OPEN
severity: Low
category: bug
tags: [material, mgir, vertex-color, translucent, OpacityFloor, naming, example-content, pwmodel]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# The name says lower bound; the graph is an upper bound

The material is generated from checked-in text, not shipped as a binary — 
`Examples/mgir/M_PwModelExample_VertexColorTranslucent.mgir`:

- `:47` — `%opacityFloor = call \`/Script/Engine.MaterialExpressionScalarParameter\`(ParameterName: "OpacityFloor", Group: "Surface", SortPriority: 32, DefaultValue: 0.35)`
- `:52` — `%opacity = call \`/Script/Engine.MaterialExpressionMultiply\`(A: %vertexColor[4], B: %opacityFloor)`
- `:58` — `output Opacity: %opacity`

`MaterialExpressionMultiply`, not `MaterialExpressionMax`; a grep of `Examples/mgir/` finds no `Max`
expression in this document. With `%vertexColor[4]` = alpha ∈ [0,1] and a `.pwmodel` `color=`
defaulting alpha to 1, the **maximum** achievable opacity is exactly 0.35, and
`color=(r,g,b,0.5)` halves it to 0.175 — the opposite of what a floor guarantees.
`MI_PwModelExample_Glass` sets it to 0.35 (`Docs/wiki-src/model.vertex-color.md:53`), so every glass
and quartz surface in the corpus is capped at 35% opacity.

## Refinement: the multiply is deliberate and documented

The report framed this as the parameter misleading by 3x with the wiring at fault. The wiring is
intentional and written down twice:

- `Examples/mgir/M_PwModelExample_VertexColorTranslucent.mgir:4-6` — "Opacity = VertexColor.A *
  OpacityFloor. A .pwmodel color= defaults its alpha to 1, so an author who never touches alpha still
  gets OpacityFloor; an author who writes color=(r,g,b,0.5) gets half of it."
- `Examples/pwmodel/gothic_window.pwmodel:80-81` — "master computes 'Opacity = VertexColor.A *
  OpacityFloor', and the instance sets OpacityFloor = 0.35. It is a MULTIPLIER, not a floor".

So this is a **naming** defect against correct adjacent documentation, not a wiring bug. The design is
a ceiling expressed as a floor, and the only reader it misleads is one who does not scroll two lines
up.

**Fix:** rename the parameter to `OpacityScale` or `MaxOpacity`, or change the expression to a `Max`
and keep the name. The rename is the cheaper and more honest of the two, and it touches example
content only.

## Scope note

The compiled instance lives in the **host project** at
`Content/PinWrightExamples/Materials/MI_PwModelExample_Glass.uasset`, not in the plugin (the plugin's
`Content/` is Python only). The MGIR text is the shipped source of truth; the `.uasset` bytes were not
opened, so this ticket does not claim the built instance carries 0.35 rather than a post-compile
override — only that the source and the docs table both say 0.35.

## History
- `#1-multiply-named-floor` `OPEN` reporter — `M_PwModelExample_VertexColorTranslucent.mgir:52` wires `Opacity` through `MaterialExpressionMultiply(A: %vertexColor[4], B: %opacityFloor)`, with the parameter declared at `:47` as `OpacityFloor`, `DefaultValue: 0.35`, and the output bound at `:58`. It is a multiply, not a `Max` — no `Max` expression exists anywhere in `Examples/mgir/` for this document — so the parameter is a ceiling: with alpha at its `.pwmodel` default of 1 the maximum opacity is 0.35, and `color=(r,g,b,0.5)` halves it to 0.175, which is the opposite of what a floor guarantees. `MI_PwModelExample_Glass` sets it to 0.35 (`Docs/wiki-src/model.vertex-color.md:53`), capping every glass and quartz surface in the corpus at 35%. Refinement to the report: the multiply is deliberate and already documented as a multiplier in two places — `M_PwModelExample_VertexColorTranslucent.mgir:4-6` and `Examples/pwmodel/gothic_window.pwmodel:80-81` ("It is a MULTIPLIER, not a floor") — so this is a naming defect against correct adjacent documentation, not a wiring bug. Fix: rename to `OpacityScale`/`MaxOpacity`, or switch the expression to `Max` and keep the name. Scope note: the compiled instance lives in the host project at `Content/PinWrightExamples/Materials/MI_PwModelExample_Glass.uasset`, outside this repo; its bytes were not opened, so no claim is made that the built asset carries 0.35 rather than a post-compile override — only that the MGIR source and the docs table agree on it.
