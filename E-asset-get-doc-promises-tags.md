---
id: E-asset-get-doc-promises-tags
title: "asset.get's registered summary promises 'asset registry tags' but the result omits the tags field — agents read the doc, expect a tags map, and detour off the real getters (asset.get_metadata / asset.list)"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [asset, asset-get, asset-get-metadata, registry-tags, docs, blend-mode, misleading-doc]
---

# `asset.get`'s documentation advertises "asset registry tags" it does not return

`asset.get`'s **registered handler summary** describes the verb as returning
registry tags, verbatim:

> `asset.get` — Read summary metadata (name, class, package, **asset registry
> tags**) for one asset by path. Loads the asset to populate fields; for a deep
> dump use asset.dump or asset.dump_folder instead.

This summary string is the single source of truth — it lives in C++ at
`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp`
(the `REGISTER_RPC_HANDLER("asset.get", ...)` line) and is the text that the
generated wiki page (`wiki-generated/asset.get.md`) and the auto-built `##
Methods` index on the `asset` namespace page both render from. (Note: the
hand-authored overlay `Docs/wiki-src/asset.md` does **not** contain the phrase
"asset registry tags"; an earlier draft of this ticket mis-cited `asset.md:65`
and a nonexistent `asset.get.md` source file — those are generated artifacts,
not the editable origin of the claim.)

But the actual result contains **no tags field at all** — only
`name` / `path` / `class` / `packagePath`:

```
call("asset.get", {assetPath:"/Game/ExampleContent/Blueprint_Communication/Materials/M_Light_Bulb_Glass"})
→ {"success":true,"result":{
     "name":"M_Light_Bulb_Glass",
     "path":"/Game/ExampleContent/Blueprint_Communication/Materials/M_Light_Bulb_Glass.M_Light_Bulb_Glass",
     "class":"/Script/Engine.Material",
     "packagePath":"/Game/ExampleContent/Blueprint_Communication/Materials"}}
```

The "asset registry tags" the doc names are simply absent. This is the
misleading half: the prose promises a field the response never carries. (The
same shallow `{name, class, ...}` shape is also visible verbatim in the repro
of the unrelated ticket `B-get-dependencies-object-path-empty` #72.)

The tags DO exist and ARE reachable — just from a different verb. The sibling
`asset.get_metadata` returns them as a typed map, including the value an agent
most often wants off a material:

```
call("asset.get_metadata", {assetPath:".../M_Light_Bulb_Glass"})
→ {"success":true,...,"tags":{
     "ShadingModel":"MSM_DefaultLit","BlendMode":"BLEND_Additive",
     "MaterialDomain":"MD_Surface","ShadingModels":"(ShadingModelField=2)",
     "TranslucencyLightingMode":"TLM_Surface", ...}}
```

So `BlendMode` (e.g. `BLEND_Additive`) is a **one-shot typed read** via
`asset.get_metadata`. `asset.list` likewise returns the tag *names* (it lists
`"BlendMode"` among an asset's `tags` array). Only `asset.get` — the verb whose
own doc advertises "asset registry tags" — drops them.

## Why this is concretely misleading (not just a missing nicety)

An agent auditing material blend modes reads the `asset.get` doc, sees "asset
registry tags," and reasonably expects `asset.get` to surface `BlendMode`. When
the result has no tags, the agent does not conclude "wrong verb, try
`asset.get_metadata`" — the doc gave it no reason to look elsewhere — so it
detours to scraping `asset.dump`'s `properties.json` / `material_instance.json`
sidecars for the blend mode instead. That is exactly the friction recorded in
the audited task (verbatim): *"neither asset.get nor asset.get_material_stats
surfaces a material's blend_mode ... the second-source blend-mode confirmation
required asset.dump's properties.json/material_instance.json sidecars rather
than a single-shot typed getter — the blend mode is a registry tag yet not
exposed by the obvious typed reads."* The capability is **not** missing
(`asset.get_metadata` is the single-shot typed getter); the doc just points the
agent at the wrong verb and overstates that verb's output.

(Note: `asset.get_material_stats` is correctly scoped and correctly documented —
"shading model, samplers, etc." — and returns `{shadingModel, instructionCount,
samplerCount}`; it never claims to return blend mode, so it is not misleading.
The misleading text is solely `asset.get`'s "asset registry tags.")

## Fix

Two valid resolutions (ergonomic, doc-or-behavior, not a correctness bug):
- **Behavior fix (preferred, chosen):** include the asset-registry `tags` map in
  `asset.get`'s result. The data is already in hand — `asset.get`'s body already
  holds the `FAssetData` (it calls `FindAssetData` to populate name/class/path),
  so emit `AssetData.TagsAndValues` as a typed name->value `tags` object exactly
  the way `asset.get_metadata` does (`AssetMetadataHandler.cpp` `tags` block) and
  `asset.list` does (the per-asset `tags` array). This makes the existing summary
  *true* and carries blend mode / NaniteEnabled on the obvious "read one asset"
  verb in one shot.
- **Doc fix (alternative):** strike "asset registry tags" from the `asset.get`
  **registered summary string** at
  `Source/.../Handlers/Asset/AssetManageHandler.cpp` (the `REGISTER_RPC_HANDLER`
  line — NOT the `asset.md` overlay, which doesn't carry the phrase) and point
  readers at `asset.get_metadata`. This re-aligns the doc to the real
  `{name, class, package, path}` output.

Either resolves the misdirection. The behavior fix is preferred because it
closes the gap at the verb the doc already points agents at, and is the only
arm with a meaningful regression test (the doc-only arm has no behavior to pin).

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a material-audit struggle task
  (seed `python.execute`; the friction was a neighbor, `asset.get`). Replay-confirmed
  on `/Game/ExampleContent/Blueprint_Communication/Materials/M_Light_Bulb_Glass`:
  `asset.get` returns `{name, path, class, packagePath}` with **no** tags field,
  despite its doc (`asset.md:65`, `asset.get.md:7`) saying it returns "asset
  registry tags"; `asset.get_metadata` on the same asset returns
  `tags:{... "BlendMode":"BLEND_Additive" ...}`, the one-shot typed blend-mode
  read the audited agent never found because the `asset.get` doc steered it
  wrong (it detoured to `asset.dump` sidecars instead). `asset.list` returns the
  tag *names* including `"BlendMode"`. Not a dup of `E-asset-path-vs-assetpath-list-drift`
  (that is `path` vs `assetPath` param naming) nor `B-asset-dump-mic-no-material-instance-json`
  (MIC dump sidecar). Ergonomic / misleading-doc, gated on the quotable
  doc-vs-output mismatch above.
- `#2-recurred-nanite-readback` `OPEN` reporter — Cross-task evidence: same
  `asset.get`-returns-no-tags friction recurred in a DemoRoom Nanite
  enable-and-verify audit (seed `render.nanite_rebuild_mesh`). The agent needed
  `NaniteEnabled` (not BlendMode this time), called `asset.get` on SM_VizButton,
  got only `name/path/class/packagePath` (friction note verbatim: "asset.get
  returns only name/path/class/packagePath and surfaces NO tag VALUES … the
  documented 'asset.get reports NaniteEnabled' round-trip isn't directly
  achievable"), and again detoured to `asset.dump` `properties.json` for the
  value. Confirms this is field-agnostic (not BlendMode-specific) — any agent
  reading the `asset.get` doc for a registry tag value is misdirected. The
  describe-side companion (static_mesh.describe also advertising and omitting
  Nanite) is filed separately as `E-static-mesh-describe-doc-promises-nanite`.
- `#3-reword-and-behavior-fix` `IN-REVIEW` fuzz3 — REWORDED then implemented the
  **behavior fix**. Reword: the false "asset registry tags" claim lives ONLY in
  the C++ registered summary string (`AssetManageHandler.cpp` asset.get
  `REGISTER_RPC_HANDLER`), not in `asset.md`/`asset.get.md` — the prior
  `asset.md:65` / `asset.get.md:7` citations were generated wiki artifacts, not
  the editable origin; title/Fix section corrected to target the summary string
  and to mark the behavior fix as the chosen arm. Fix (code): `asset.get` now
  emits `AssetData.TagsAndValues` as a typed name->value `tags` object on its
  result, mirroring `asset.get_metadata` and `asset.list` — the doc is now true
  and BlendMode/NaniteEnabled are a one-shot typed read off the obvious verb.
  Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetManageHandler.cpp`
  (asset.get body, added the tags map). Regression test:
  `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAssetGetTags.cpp`
  (`FAssetGetReturnsRegistryTagsTest`,
  `EditorAutomationRpcGateway.asset.get.ReturnsRegistryTags`) — builds a real
  in-memory Blueprint, registers it with the asset registry, dispatches
  `asset.get` through the production dispatcher, and asserts the result carries a
  non-empty `tags` map; reverting the fix drops `result.tags` and fails it.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
