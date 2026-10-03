---
id: E-material-layer-stack-readback-objectpath-form
title: "material.authoring layer-stack verbs (create_material_layer, create_material_layer_blend, set_material_layer_stack) have no workflow section in the wiki"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-authoring, material-layers, get-material-node-details, set-material-layer-stack, object-path, readback, path-form, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
rice: [1, 1, 1, 1]
priority: 8
---

# The material-layer authoring verbs have no workflow section in the material.authoring wiki

`material.authoring.create_material_layer`, `create_material_layer_blend` and
`set_material_layer_stack` (`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:4213`,
`:4271`, `:4331`) have no H3 and no workflow in `docs/wiki-src/material.authoring.md`. They
appear only in passing lists: the `AMBIGUOUS_NODE` note (`:53`), the no-push list (`:58`), and
the `name`/`path` contract for creators (`:76`, `:175`). A caller building a layered material
gets no help from the wiki on the order of calls or on the stack verb's arguments:

- The stack is wired by creating the layer and blend assets, then
  `add_material_node`/`add_expression` with class `MaterialAttributeLayers`, then
  `set_material_layer_stack`, then reading it back with `get_material_node_details`.
- `set_material_layer_stack` addresses the node through **`expressionId`** (a GUID or object
  name, resolved at `:4367-4370`), not `nodeId`, which the `:53` list implies. `blends` must
  have exactly `layers.length - 1` entries, otherwise `INVALID_PARAMS` (`:4359-4365`).
- `seedTemplate` (default true) seeds the validation-passing default body, so a new layer or
  blend compiles as created (`:4221`, `:4279`).
- `get_material_node_details` reports layer and blend references in object-path form
  (`/Game/.../ML_RockBase.ML_RockBase`), the plugin-wide reference form. A caller comparing
  them with the package paths it passed must compare by package.

**Fix:** Add a `### Material layers` section to `docs/wiki-src/material.authoring.md` that covers
the four points above, plus the fact that `set_material_layer_stack` does not push to
landscapes (finish with `compile_material`, per `:58`).

**Acceptance:** `material.authoring.md` has a section naming all three verbs, the
`MaterialAttributeLayers` node step, the `expressionId` slot, the `blends = layers - 1` rule and
the object-path readback form. The generated wiki page for `set_material_layer_stack` links to it.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the
  `material.authoring.create_material_layer` layered-terrain task (focus
  `material.authoring.create_material_layer`, outcome `clean`; all 8 execute
  calls — 2 `create_material_layer`, 1 `create_material_layer_blend`,
  `create_material`, `add_material_node`, `set_material_layer_stack`,
  `get_material_info`, `get_material_node_details` — succeeded first try, no
  retries, no workarounds, judge filed nothing). Distinct PROCESS angle: the
  single friction the attempt named, verbatim — *"the only minor note is
  get_material_node_details returns layer/blend paths in full object-path form
  (e.g. `.ML_RockBase` suffix), which still matches the requested asset paths."*
  The caller created/wired the assets in package-path form
  (`/Game/Materials/Layers/ML_RockBase`) but the readback echoes object-path form
  (`/Game/Materials/Layers/ML_RockBase.ML_RockBase`), so the step-6 success check
  ("confirm the node references both layers + the blend") required caller-side
  `.AssetName`-suffix normalization to assert equality. Same write-vs-read
  path-form family as the DONE `B-widget-resourceobject-path-prefix-inconsistent`
  (normalize asset refs to one canonical form); distinct from the input-side
  `B-get-dependencies-object-path-empty` and the field-coverage
  `B-material-get-node-details-missing-pins-props`. Dedup: ripgrep across OPEN +
  closed found no ticket on the layer-stack readback path-form. Fix: emit the
  package-path form (symmetric with the layer-stack RPCs' input), or document the
  object-path form; plus add the missing layer-stack section to
  `docs/wiki-src/material.authoring.md`.
- `#2-rephrased` `OPEN` developer — Retitled and rescoped to the docs gap. Dropped the ask to normalize the readback to package paths: the object-path (`pkg.Asset`) form is the plugin-wide reference form, the original reporter noted that it "still matches", and the form now gets one doc sentence. The old text said the overlay had no section at all. In fact the verbs are named in passing at `material.authoring.md:53`, `:58`, `:76`, `:175`, but there is still no H3 or workflow. Added the `expressionId` slot (`MaterialAuthoringHandler.cpp:4367-4370`), the `blends = layers - 1` rule (`:4359-4365`) and `seedTemplate`. Severity unchanged (Low, docs). RICE unchanged.
