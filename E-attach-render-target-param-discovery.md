---
id: E-attach-render-target-param-discovery
title: "render.attach_render_target_to_volume gives no path to discover a valid parameterName (or the material contract), forcing a multi-dump material hunt"
status: OPEN
severity: Low
category: ergonomic
tags: [render, render-target, post-process, undocumented-material-contract, docs]
encounters: 1
costly: 1
lastSeen: 2026-07-10T22:13:47+0300
---

# attach_render_target_to_volume has no discovery path for its parameterName

`render.attach_render_target_to_volume` takes a required-in-practice
`parameterName` (plus `materialPath`) and binds the render target to that
texture parameter of a MID it creates and adds as a WeightedBlendable. But
its wiki page states neither:

- **(a) the material contract** — that `materialPath` must be a
  post-process-domain material that actually **exposes a texture parameter**
  (a `TextureSampleParameter2D` / subclass such as `AntialiasedTextureMask`),
  not just any post-process showcase material; nor
- **(b) how to enumerate that material's texture parameter names** so the
  caller can pass a real one.

With no guidance, the attempt hand-rolled a material-parameter hunt. Many of
this project's post-process showcase materials expose **no** texture parameter
(they hardcode `SceneTexture` or expose only scalar/vector params), so the
caller probed four candidate materials with `asset.dump` (a heavy call that
writes `meta.json` / `properties.json` / `mgir.txt` to disk each time),
Grep'd each dumped `mgir.txt` for parameters ("No matches found"), and finally
`Bash`-grepped the installed engine C++ headers
(`MaterialExpressionAntialiasedTextureMask.h`) to confirm that
`AntialiasedTextureMask` subclasses `UMaterialExpressionTextureSampleParameter2D`
— i.e. that the auto-generated expression name
`MaterialExpressionAntialiasedTextureMask_0` is in fact a settable texture
parameter. Four disk-writing dump RPCs + several Greps + an engine-source read,
all to discover one `FName`.

A purpose-built helper, `material.authoring.get_material_info`, already returns
a material's parameters inline in one RPC (no disk writes, no Grep), but the
discovery path from the attach method never points there.

## What it should do

`render.attach_render_target_to_volume`'s wiki overlay should:

1. State the material contract: `materialPath` must be a post-process-domain
   material that exposes a texture parameter (name the common source —
   `TextureSampleParameter2D` and subclasses like `AntialiasedTextureMask`),
   and note that the method creates a MID and adds it as a WeightedBlendable.
2. Cross-reference `material.authoring.get_material_info` as the one-RPC way to
   list a material's texture parameter names, collapsing the discovery cost from
   four disk-writing `asset.dump`s + Greps + an engine-source read to one info
   call per candidate.

Wiki page to improve: `docs/wiki-src/render.md` (the
`attach_render_target_to_volume` section) and the per-method overlay
`docs/wiki-src/render.attach_render_target_to_volume.md` the attempt actually
read.

Note: the attach RPC itself was clean (one shot, `attached:true`) and the goal
succeeded; part of the hunt is inherent content variance (few of this project's
post-process materials expose a texture parameter). This is a
discoverability/docs gap, not a functional defect.

## Relationship to B-attach-render-target-phantom-param

Same method, distinct root cause — file separately, do not merge.
`B-attach-render-target-phantom-param` (the Judge's replay-confirmed bug) is
the **functional** silent-false-success: the method returns `attached:true`
even for a nonexistent parameter, so a caller who guesses the auto-generated
expression name wrong gets an inert binding with no error. THIS ticket is the
**up-front discoverability** gap: even before that bug bites, the wiki gives no
way to find a valid parameter name or to know the material must expose one. The
two compound each other — a caller with no enumeration path is exactly the
caller who then passes a phantom name — but the fixes differ (docs/cross-ref
here; parameter-existence validation there).

## Evidence

From the CallAnalyzer trace (15 pinwright RPCs, zero errored/retried calls,
goal self-verified). The parameter-discovery hunt before the single clean
attach call:

- `asset.dump` PPMAT_BlendablePostProcess — no texture param
- `asset.dump` M_PostProcessBlendable — no texture param (hardcoded SceneTexture)
- `asset.dump` M_ReflectionDemo_MAT — only scalar/vector params (Roughness/Color/Metallic)
- `asset.dump` PPMAT_BlendablePostProcess2 — has the tex-sample param, but named `MaterialExpressionAntialiasedTextureMask_0`
- plus repeated Greps of the dumped mgir ("No matches found") and a Bash grep of the engine C++ header to confirm the parameter type.

Attempt friction note (verbatim): "the two obvious 'BlendablePostProcess'
post-process showcase materials and the M_ReflectionDemo base material have NO
texture parameter ... so finding a post-process showcase material with a real
settable texture parameter required dumping 4 materials before landing on
PPMAT_BlendablePostProcess2, whose texture param comes from an
AntialiasedTextureMask (a TextureSampleParameter2D subclass) with an
auto-generated name."

severity rationale: impact=docs/discoverability (Low) x reach=rare (one
specialized render-target->PPV method) -> Low

## History

- `#1-initial-audit` `OPEN` reporter — CallAnalyzer flagged one inefficiency (workaround/discoverability) on the focus method `render.attach_render_target_to_volume`: no path to discover a valid `parameterName` or the material-exposes-a-texture-parameter contract, so the attempt spent ~4 disk-writing `asset.dump` RPCs + several Greps + an engine-source header read to find one FName, when `material.authoring.get_material_info` returns a material's parameters inline in one RPC. Docs/discoverability gap; the attach RPC itself was clean and the goal succeeded. Distinct root cause from `B-attach-render-target-phantom-param` (functional silent-false-success on the same method). Wiki pages to improve: `docs/wiki-src/render.md` + `docs/wiki-src/render.attach_render_target_to_volume.md`.
