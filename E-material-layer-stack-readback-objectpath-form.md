---
id: E-material-layer-stack-readback-objectpath-form
title: "get_material_node_details echoes MaterialAttributeLayers layer/blend refs in object-path form (/Game/.../ML_RockBase.ML_RockBase) while the caller created/requested them in package-path form — success-check needs caller-side normalization"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-authoring, material-layers, get-material-node-details, set-material-layer-stack, object-path, readback, path-form, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# Layer-stack readback returns object-path-suffixed asset paths, not the package paths the caller wrote

The just-shipped material-layers surface (`F-material-layers-asset-authoring`,
DONE — `create_material_layer` / `create_material_layer_blend` /
`set_material_layer_stack`) round-trips cleanly, but the **readback form is
inconsistent with the write form**. The caller creates the layer/blend assets
with, and wires the stack from, **package-path** strings:

- `create_material_layer {name:"ML_RockBase", path:"/Game/Materials/Layers"}`
- `set_material_layer_stack {layers:[".../ML_RockBase", ".../ML_MossOverlay"], blends:[".../MLB_RockToMoss"]}`

…but when the caller reads the wired node back with
`get_material_node_details` to confirm the stack took, the layer/blend
references come back in **object-path** form — the package path plus a
`.AssetName` object suffix, e.g. `/Game/Materials/Layers/ML_RockBase.ML_RockBase`
rather than the `/Game/Materials/Layers/ML_RockBase` the caller passed in.

This is a (Low) ergonomic readback-consistency gap, not a tool bug: the stack
*was* wired correctly and the readback *does* reference the right assets. The
friction is that the success check ("confirm the MaterialAttributeLayers node now
references both layer assets and the blend asset") becomes a string compare
between two different path **forms**, so the caller must normalize the `.AssetName`
suffix (strip the trailing `.Object`, or `FSoftObjectPath::GetLongPackageName`)
before asserting equality against the paths it created — exactly the same
write-vs-read path-form normalization tax that the DONE
`B-widget-resourceobject-path-prefix-inconsistent` ticket eliminated for tree.xml
`ResourceObject` references by normalizing to one canonical form.

## What it should do

Pick one canonical form for asset references in `get_material_node_details`'s
layer/blend readback and emit it consistently, so the success check is a direct
string equality against the paths the caller created/wired. The natural choice is
the **package-path** form the layer-stack RPCs already accept on input
(`/Game/Materials/Layers/ML_RockBase`), making write and read symmetric. If the
object-path form is deliberate (e.g. to disambiguate which object inside the
package), then at minimum document it (see Docs angle) so callers strip the
suffix before comparing rather than discovering the mismatch at verify time.

Either way the readback should not silently change the path **form** between what
the caller wrote and what it reads back, on a brand-new authoring surface whose
whole point is round-trip layer-stack construction.

This is distinct from the two nearby path-form tickets:
- `B-get-dependencies-object-path-empty` (IN-REVIEW) — the **input** side: an
  object-path `assetPath` is silently mishandled at lookup. This ticket is the
  **output** side: the readback emits the object-path form the caller never
  asked for.
- `B-material-get-node-details-missing-pins-props` (DONE) — added the layer/blend
  reference fields to `get_material_node_details` in the first place; this is the
  follow-on consistency note on the **form** those fields are emitted in.

## Docs angle (`docs/wiki-src/material.authoring.md`)

The overlay page has **no section at all** for the layer-stack surface — it
predates the `F-material-layers-asset-authoring` feature, so `create_material_layer`,
`create_material_layer_blend`, `set_material_layer_stack`, and the
`MaterialAttributeLayers` `get_material_node_details` readback are undocumented.
Add a `### Material layers` (or per-RPC H3) section that (a) documents the create
→ `add_material_node MaterialAttributeLayers` → `set_material_layer_stack` →
`get_material_node_details` workflow, and (b) notes that the readback reports
layer/blend refs in object-path (`pkg.AssetName`) form, so callers verifying
against the package paths they passed in must normalize the suffix before
comparing. Page to improve: `docs/wiki-src/material.authoring.md`.

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
