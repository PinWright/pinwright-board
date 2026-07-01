---
id: E-create-material-instance-duplicate-divergent-shape
title: "Two create_material_instance RPCs (asset.* legacy vs material.authoring.*) with divergent param shapes; the self-described legacy one is the one a list-driven agent lands on, and only it supports override-at-create"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [material, material-instance, create_material_instance, duplicate-method, param-shape-drift, asset, docs]
---

# Two `create_material_instance` verbs, divergent shapes, "preferred" one lacks the inline-override convenience

There are two registered RPCs that create a `UMaterialInstanceConstant`:

- `asset.create_material_instance` (`AssetMaterialHandler.cpp:108`) — its own
  description says *"For new code prefer `material.authoring.create_material_instance`;
  this entry remains as the **legacy creator**."* Params:
  `name` (req), `path` (**req**), `parentMaterial` (req),
  **`parameters` (opt object — inline scalar/vector/texture overrides at creation)**.
- `material.authoring.create_material_instance` (`MaterialAuthoringHandler.cpp:1533`)
  — the self-declared preferred verb. Params: `name` (req),
  `parentMaterial` (req), `path` (**opt**, default `/Game/Materials`),
  `save` (opt). Its description: *"Use the `set_*_parameter_value` methods
  afterward to override parameters."* — i.e. **no inline `parameters` slot**.

Two frictions stack on this duplication:

**1. The "preferred" verb is the less capable one.** The legacy
`asset.create_material_instance` can set overrides in the same call via its
`parameters` object (one round-trip: create + tint). The preferred
`material.authoring.create_material_instance` cannot — it forces a separate
`set_*_parameter_value` / `set_material_instance_parameters` call afterward.
So an agent that follows the "prefer material.authoring" steer pays an extra
step the legacy verb would have saved, and an agent that wants the one-shot
convenience is pushed onto a verb the plugin itself labels legacy. The
deprecation arrow points away from the more ergonomic surface.

**2. Divergent param shape between the two creators.** Same conceptual verb,
different contracts: `path` is **required** on the asset verb but **optional**
(defaulted) on the authoring verb; the asset verb takes `parameters`, the
authoring verb takes `save`. An agent that learns one shape and switches verbs
(e.g. discovers the asset creator next to `asset.list_material_instances`, then
moves to `material.authoring.*` for the param setters) carries the wrong
mental model across the boundary. This is the same cross-verb / cross-namespace
shape-drift the create-slot tickets argue against
(`E-material-create-combined-assetpath-split` for the `name`+`path` split,
`E-asset-path-vs-assetpath-list-drift` for "list teaches one spelling, the next
verb rejects it").

This is not a tool bug — both verbs work — and in the audited task there were
zero errors. It is a quiet discoverability + duplicate-surface ergonomic cost
that shows up even on a clean run: a list-driven material-instance task reaches
into `asset.*` to create (the legacy verb, because `list_material_instances`
also lives in `asset.*`), then into `material.authoring.*` to set params and
read back. The friction is invisible in the call log (everything is `ok`) but
real: the agent did create-then-set as two calls when the legacy creator's
inline `parameters` would have done it in one, and it built on the verb the
plugin asks new code not to use.

## What it should do

Pick one canonical creator and make it the capable one. Either:

1. **Add the `parameters` inline-override slot to
   `material.authoring.create_material_instance`** so the preferred verb is a
   strict superset of the legacy one. Adopt the **type-keyed batch shape that
   `material.authoring.set_material_instance_parameters` already establishes** —
   `{scalar:{Name:num}, vector:{Name:{r,g,b,a}}, texture:{Name:assetPath},
   staticSwitch:{Name:bool}}` — *not* the asset verb's slightly different
   `{scalar, vector, texture}` map (which also lacks `staticSwitch`), so the
   inline create-override and the batch setter speak one shape. Reuse the batch
   verb's apply path (one `FMaterialInstanceParameterUpdateContext`). This closes
   both the capability gap and the shape drift at once. (The legacy asset verb
   can stay as a thin alias / be left in place; the point is the preferred verb
   becomes the capable one.)
2. Or, if the asset verb is the keeper, drop the "prefer material.authoring"
   steer from its description and instead deprecate the authoring duplicate.

Either way the two should not advertise *opposite* preferences while diverging
on `path` required-ness and on whether override-at-create exists.

## Docs angle (`docs/wiki-src/asset.md`, `docs/wiki-src/material.authoring.md`)

Until the verbs are unified, the overlays should make the split explicit so a
list-driven agent doesn't have to read both method pages to discover which
creator to use and what each accepts:

- `docs/wiki-src/asset.md`: near `create_material_instance` /
  `list_material_instances`, note that `asset.create_material_instance` is the
  legacy creator, that it (uniquely) accepts an inline `parameters` override
  object, and that `path` is required here (vs optional on the authoring verb).
- `docs/wiki-src/material.authoring.md`: note that
  `material.authoring.create_material_instance` has **no** inline `parameters`
  slot — overrides must follow via `set_material_instance_parameters` — so an
  agent expecting override-at-create reaches for the right verb (or the right
  two-call sequence) on the first try.

The downstream wiki edit is the wiki-authoring process, not this audit; this
ticket records the gap and names the pages.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a clean
  `asset.list_material_instances` task (focus `asset.list_material_instances`,
  outcome `clean`, self-reported friction "none" — 26 calls, zero retries/errors;
  the judge filed nothing). Distinct PROCESS angle surfaced from the call log:
  the agent created both new MICs (`MI_Metal_Brushed_Gold`, `MI_Metal_Dark_Steel`)
  via `asset.create_material_instance` — the verb whose own description
  (`AssetMaterialHandler.cpp:108`) says "prefer `material.authoring.create_material_instance`;
  this entry remains as the legacy creator" — then ran `set_material_instance_parameters`
  (`material.authoring.*`) as a separate step for each. The legacy verb it used
  actually exposes an inline `parameters` override object that would have done
  create+override in one call, while the plugin-preferred
  `material.authoring.create_material_instance` (`MaterialAuthoringHandler.cpp:1533`)
  omits that slot and also flips `path` from required to optional — so the two
  duplicate creators advertise opposite preferences while diverging on shape and
  capability. Source-verified both registrations and param specs. Low severity
  (both work; no error, no blocked progress) — E-/docs because it is a
  duplicate-surface + cross-verb shape-drift + "preferred verb is less capable"
  ergonomic gap, not a wrong result. Distinct from
  `E-material-create-combined-assetpath-split` (that ticket is the `name`+`path`
  vs combined-`assetPath` split on the create verbs; this one is the two-verbs /
  divergent-shape / missing-inline-`parameters` duplication) and from
  `F-material-instance-overrides-incomplete` (IN-REVIEW — that added the
  `set_material_instance_parameters` batch read/write surface but did not unify
  or reconcile the two create verbs).
- `#2-fix` `IN-REVIEW` developer — Implemented proposed-fix option 1 (and the
  docs angle). Made the **preferred** verb the capable one:
  `material.authoring.create_material_instance` now accepts an optional inline
  `parameters` object in the type-keyed `{scalar, vector, texture, staticSwitch}`
  shape that `set_material_instance_parameters` already establishes, so
  create+override is one round-trip and both verbs speak one shape (closes the
  capability-vs-deprecation inversion + shape drift the ticket leads with).
  Extracted the batch verb's apply logic into a shared file-local helper
  `ApplyMaterialInstanceParameterOverrides` (one `FMaterialInstanceParameterUpdateContext`,
  single recompile) so the create verb and `set_material_instance_parameters`
  share one apply path; the batch verb now re-nests its four top-level type maps
  and delegates (wire contract unchanged), and the now-dead `JsonToLinearColor`
  helper was removed. Reworded the **Fix:** option-1 shape description (it had
  mis-stated the convergent shape as a flat name→value map with 3–4 array
  vectors; the real convergent shape is the type-keyed batch shape) and corrected
  the stale `F-` "DONE"→"IN-REVIEW" cross-reference. Files:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Material/MaterialAuthoringHandler.cpp`,
  `docs/wiki-src/material.authoring.md` (Instance API note + new
  `### material.authoring.create_material_instance` H3). Regression test:
  `Source/EditorAutomationRpcGateway/Private/Tests/Material/TestMaterialInstanceOverrides.cpp`
  — `FMaterialInstanceCreateWithInlineParamsTest` drives the real create handler
  with `parameters={scalar:{Roughness:0.9}}` and asserts via
  `get_material_instance_info` that `overrides.scalar.Roughness == 0.9` landed at
  creation (fails if the inline-override slot is reverted). Did not compile/run
  (later phase).
- `#3-recurs-on-clean-material-task` `OPEN` reporter — Additional evidence (struggle
  audit, different task): the same create-then-set multi-call friction recurs even with
  the fix authored. A clean `material.authoring` task (build `M_WetFloor_Master` +
  two instances `MI_WetFloor_Glossy` / `MI_WetFloor_Matte`; outcome `clean`,
  self-reported friction "none", 16 execute calls, zero errors/retries, no
  python.execute) created **both** instances via `material.authoring.create_material_instance`
  with **no inline `parameters`**, then overrode separately: Glossy = create + 
  `set_scalar_parameter_value(Roughness 0.08)` + `set_vector_parameter_value(BaseColor 0.2,0.3,0.45,1)`
  = **3 round-trips** (two typed setters → two separate recompiles) where the documented
  inline `parameters:{scalar:{Roughness:0.08},vector:{BaseColor:{…}}}` on create is **1**;
  Matte = create + `set_material_instance_parameters(Roughness 0.85, Metallic 0.0)`
  = **2 round-trips** where inline-on-create is **1**. Notably the agent's wiki-nav
  visited the `create_material_instance`, `set_scalar_parameter_value`,
  `set_vector_parameter_value`, AND `set_material_instance_parameters` pages as
  separate nav steps yet still chose the multi-call path — i.e. the one-shot
  create-and-tint convenience (now documented at `material.authoring.md` lines 62/142
  as the preferred path) is still not being reached for on first authoring even when
  the relevant pages are read. Reinforces #2's fix and the docs angle: the convenience
  exists but discoverability of "set overrides *at create*" remains weak; a tester
  verifying the IN-REVIEW fix should also confirm the wiki H3 / Instance-API note make
  the inline-`parameters`-on-create path the obvious default so create-then-set stops
  being the reflexive shape. No new code requested — this is a recurrence/evidence
  append, not a re-file.
