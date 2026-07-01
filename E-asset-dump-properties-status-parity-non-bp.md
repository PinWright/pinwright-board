---
id: E-asset-dump-properties-status-parity-non-bp
title: "asset.dump meta.json propertiesStatus schema parity: emit block for non-Blueprint assets too"
status: DONE
severity: Low
category: ergonomic
tags: [asset-dump, properties, meta-json, schema-parity]
---

# asset.dump meta.json propertiesStatus schema parity: emit block for non-Blueprint assets too

Follow-up to `B-asset-dump-empty-properties-no-success-marker` (DONE). That fix
added a `propertiesStatus` block to `meta.json` for Blueprint-class entries so
consumers can distinguish intentional empty `properties.json` from a failed
dump. The block is only emitted on Blueprint-class entries, however —
Material, MaterialInstance, MaterialFunction, StaticMesh, Texture2D, and the
other non-Blueprint asset dumps have **no `propertiesStatus` block at all** in
their `meta.json` (verified across all 187 StaticMesh entries in one slice, and
similarly empty across the other non-BP asset kinds).

This forces a consumer scanning `meta.propertiesStatus.status` to disambiguate
three states instead of two:
1. Field present with a known value (current BP path)
2. Field absent because the asset type intentionally has no property dump
3. Field absent because the dumper failed before writing meta

States 2 and 3 are indistinguishable from a single uniform check, which is the
exact ergonomic problem the parent ticket set out to fix — solved for BPs,
unsolved for everything else.

**Affected asset types (non-exhaustive):** Material, MaterialInstance,
MaterialFunction, StaticMesh, SkeletalMesh, Texture, Texture2D, SoundCue,
SoundWave, DataTable, NiagaraSystem, NiagaraEmitter, LevelSequence, MetaSound
— anything whose property surface lives in dedicated sidecars
(`static_mesh.json`, `material_instance.json`, `mgir.txt`, `niagara_system.json`,
etc.) rather than `properties.json`.

**Recommendation:** Emit a uniform `propertiesStatus` block on every dumped
asset's `meta.json`. For asset kinds that don't participate in
`properties.json`, write:

```json
"propertiesStatus": { "status": "n/a", "reason": "non_blueprint_asset" }
```

That gives consumers a single check: `meta.propertiesStatus?.status` is one of
`"ok"`, `"empty"`, `"n/a"`, or `"error"` — absent always means the dumper
failed.

**Workaround:** Branch the consumer-side check on `meta.className` /
`meta.parentClass` to decide whether `propertiesStatus` is expected. Brittle
when new asset types are added.

**Fix:** In the asset-dump meta builder, add a default
`propertiesStatus = { status: "n/a", reason: "non_blueprint_asset" }` write
on the non-Blueprint code paths (Material, Texture, StaticMesh, etc.) so the
field is always present on a successful dump regardless of asset kind.

## History
- `#1-initial-report` `OPEN` reporter — Schema-parity gap: `propertiesStatus` block in `meta.json` is only emitted on Blueprint-class entries after the B-asset-dump-empty-properties-no-success-marker fix. Non-BP assets (Material, MaterialInstance, StaticMesh, Texture2D, MaterialFunction, ...) have no `propertiesStatus` block at all, leaving consumers unable to distinguish "asset type intentionally has no properties dump" from "tool error" with a single uniform check. Recommend emitting `{ status: "n/a", reason: "non_blueprint_asset" }` on non-BP meta so a single `meta.propertiesStatus?.status` check covers all asset kinds.
- `#2-non-bp-n-a-default` `IN-REVIEW` developer — In `Handlers/Asset/AssetDumpHandler.cpp::BuildAllFilesForAsset`, after the type-dispatch chain runs and before `AddMetaFile()`, default `BlueprintPropertiesStatus` to `{ Status: "n/a", Reason: "non_blueprint_asset" }` when the BP path didn't populate it (i.e. the asset went through any non-Blueprint branch — Material, Texture, StaticMesh, Niagara, DataTable, etc.). `AttachBlueprintPropertiesStatusToMeta` now unconditionally annotates `meta.json` so consumers can read `meta.propertiesStatus.status` uniformly. The `UObjectRedirector` short-circuit at the top of the function also explicitly sets the same n/a marker before its early return. Bundled into the schema-v6 bump in `Utils/AssetDumpBuilder.cpp`.
- `#3-skip-editor-offline` `SKIP` tester — Editor not running and MCP gateway at :19880 refuses connections, so a live `asset.dump` on a non-BP asset can't be exercised. Cache under `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/` is still schema v5 (pre-fix), no v6 dumps anywhere to inspect. Source-side audit confirmed: `AssetDumpHandler.cpp:789-793` defaults `BlueprintPropertiesStatus` to `{n/a, non_blueprint_asset}` on the non-BP fall-through and unconditionally calls `AttachBlueprintPropertiesStatusToMeta`; `:449-454` does the same for the redirector short-circuit; `Utils/AssetDumpBuilder.cpp:146` bumps to v6 with a matching comment. Re-verify after the next editor session by re-dumping a Texture/StaticMesh/Material and checking `meta.propertiesStatus.status == "n/a"`.
- `#4-verify-non-bp-parity` `DONE` tester — Verified: live `asset.dump` on `/Game/Textures/Asset` (Texture2D) and `/Engine/VREditor/TransformGizmo/SM_Sequencer_Node` (StaticMesh) both produced `meta.json` at `dumpSchemaVersion: 6` with `propertiesStatus: { status: "n/a", reason: "non_blueprint_asset" }`. Schema-parity gap closed on the two non-BP kinds sampled.
