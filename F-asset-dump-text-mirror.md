---
id: F-asset-dump-text-mirror
title: "asset.dump / asset.dump_folder — text mirror of uasset state to disk"
status: DONE
severity: Medium
category: feature
tags: [ergonomics, inspection, persistence, mcp-client]
---

# asset.dump / asset.dump_folder — text mirror of uasset state to disk

Pulling a full picture of an asset over MCP today means chaining 3–5 calls
(`blueprint.inspect`, `blueprint.decompile`, `widget.export_xml`,
`property.list`, `asset.get_metadata`) and carrying the union of their
JSON in conversation context. It is expensive, lossy (no inheritance /
override marking), and leaves no persistent artifact to diff across edits.

This entry tracks adding two RPC methods that write a single asset's full
introspectable state to disk as separate text files
(`properties.json`, `tree.xml`, `bpir.txt`, `scs.json`, `meta.json`) plus
a bulk wrapper that mirrors a content folder. Output accumulates under
`<ProjectRoot>/.editor-automation/asset-dumps/` with package paths
mirrored 1:1, producing a greppable, diffable shadow tree.

**Scope (v1):** Blueprints, Widget Blueprints, and generic UObject assets
(DataAssets, DataTables). Materials / Niagara / Level Sequences explicitly
out of scope — they receive `meta.json` + a generic `properties.json` only
and point at their existing specialized inspectors.

**Files per asset type:**
- WBP: `meta.json`, `properties.json`, `tree.xml` (overridden-only), `bpir.txt`
- BP: `meta.json`, `properties.json`, `bpir.txt`, `scs.json` (if non-empty)
- Generic UObject: `meta.json`, `properties.json`

**properties.json shape** (per property, sorted by name):
```
{
  "type": "<CPPType>",
  "value": <serialized>,
  "inherited_from": "ParentClassName" | null,
  "is_overridden_locally": true | false,
  "flags": ["Edit", "BlueprintReadOnly", ...]
}
```

**Overwrite semantics:** each `asset.dump` purges the asset's dump directory
and rewrites applicable files atomically (`.tmp` + rename). `asset.dump_folder`
walks AssetRegistry and reconciles — prunes dump dirs whose source asset
no longer exists, scoped to the swept subtree only.

**Fix:** New `Handlers/Asset/AssetDumpHandler.cpp` orchestrates. New
`Utils/AssetDumpWriter.{h,cpp}` owns disk I/O (dump root, purge, atomic
write, long-path guard). New `Utils/AssetDumpBuilder.{h,cpp}` contains
per-aspect builders (meta, properties, bpir, widget xml, scs). Property
walker added to `Utils/PropertyUtils` with CDO-vs-parent comparison for
inheritance / override marking. Reuses `FBpirDecompiler`,
`FSCSHandlers::GetBlueprintSCS`, and `WidgetXmlExportHandler`'s internal
XML builder (extracted during this work).

Full plan: `C:\Users\Alexander\.claude\plans\no-spec-only-plan-ethereal-bird.md`.

## History
- `#1-initial-scope` `OPEN` reporter — Scoped the feature: two RPC methods (`asset.dump`, `asset.dump_folder`), layout under `.editor-automation/asset-dumps/`, separate files per aspect, inheritance-marked properties, overridden-only widget XML. Implementation in progress under the cited plan.
- `#2-implementation-landed` `IN-REVIEW` developer — Landed across 6 batches (code uncommitted, in working tree, plugin needs a rebuild before testing): (A) `Utils/AssetDumpWriter.{h,cpp}` — dump-root resolver, MAX_PATH guard, strict-subpath sanity check, per-file atomic `.tmp`-then-rename with directory purge. (B) `Utils/PropertyUtils` — `ExportPropertyToJsonValueWithInheritance` + `BuildClassPropertyJson`; CDO-vs-parent comparison via `Property->Identical` with `PPF_DeepComparison`, whitelisted CPF_* flag set, owner-class-based `inherited_from`. (C) `Utils/AssetDumpBuilder.{h,cpp}` with four builders (meta, bpir concat per-graph ordered ubergraph→function(alpha)→macro(alpha), overridden-only widget XML, SCS json). Widget XML builder needed its own TU-local copy of `BuildXmlString`/`CollectOverriddenAttributes` in a new `Handlers/UI/WidgetXmlExporter.{h,cpp}` — the existing `widget.export_xml` handler was left byte-identical on the wire to avoid regressing geometry/subtree/ambiguity support. (D) `Handlers/Asset/AssetDumpHandler.{h,cpp}` with `asset.dump` registration, exported `DumpSingleAsset`/`FDumpSingleResult` for test reuse, and a `SerializeSortedJsonObject` recursive helper that walks nested objects with sorted keys (needed because `FJsonObject::Values` is a hashed TMap, not insertion-ordered). (E) same file: `asset.dump_folder` with AssetRegistry walk + `ReconcileMirrorSubtree` pruning orphan dump dirs whose source asset no longer exists — reconciliation scoped strictly to the swept subtree, `meta.json` used as ownership marker so user files/notes survive. (F) `docs/asset-dump.md` reference + `docs/index.md` entry; two error-path tests appended (long-path, load-failure). Tests total: `TestAssetDumpWriter.cpp`, `TestAssetDumpInheritance.cpp`, `TestAssetDumpBuilder.cpp`, `TestAssetDumpHandler.cpp`. Several tests are handler-light (no heavy live fixtures) — noted in each file's comments where a simplification was taken.
- `#3-crash-on-leaf-property` `IN-REVIEW` developer — Live test on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_Race_Main` reproduced an access violation inside `FSoftObjectProperty::Identical`. Root cause: `ExportPropertyToJsonValueWithInheritance` called `Property->ContainerPtrToValuePtr(ParentContainer)` unconditionally. When a property is declared on the leaf asset class (here `ActiveHUD` on `W_HUD_Race_Main_C`), the parent CDO's memory layout does not contain that property at the computed offset — the pointer lands in unrelated memory. For non-POD types (`FSoftObjectPath`, `FText`, containers), `Identical` then copies the struct and crashes. Fix: tightened `ChildContainer` / `ParentContainer` signatures to `UObject*`, and gated the `Identical` call behind `ParentContainer->GetClass()->IsChildOf(Property->GetOwnerClass())` — skip (treat as overridden) when the parent's class does not declare or inherit the property. New lesson appended to `lessons.md`.
- `#5-verified-dump-and-folder` `DONE` tester — Verified live: (1) `mcp__editor_automation__.call path="asset.dump" args={"assetPath":"/Game/App/UI/Test/W_McpVerifyTemp"}` returned `writtenPaths: [meta.json, properties.json, tree.xml, bpir.txt], dumpDir: "C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/App/UI/Test/W_McpVerifyTemp", mode: "dump"` and `ls` confirmed all 4 files on disk (bpir.txt 1013B, meta.json 297B, properties.json 14558B, tree.xml 569B). (2) `mcp__editor_automation__.call path="asset.dump_folder" args={"folderPath":"/Game/App/UI/Test","recursive":true}` returned synchronously in <1 s with `assetCount: 3, message: "asset.dump_folder: started, ..."`, then `mcp__editor_automation__.call path="asset.dump_folder_status" args={}` showed `inProgress: false` (3 assets, finished by next tick). Both kickoff RPC and the status companion RPC behave per spec.
- `#4-async-folder-dump-rework` `IN-REVIEW` developer — Reworked `asset.dump_folder` from synchronous to fire-and-forget after live test on `/App/App/UI` (941 assets) hit the MCP client's 14.8 s RPC timeout even though the server-side sweep completed successfully. Change: handler now returns immediately in ~100 ms with `{message, rootDir, folderPath, startedAt, assetCount}`; a game-thread ticker (`FTSTicker::GetCoreTicker()`, 8 ms-per-tick budget) processes queued assets in the background; reconciliation runs in `FinalizeFolderDump` once the last asset is done. New companion RPC `asset.dump_folder_status` returns `{inProgress: false}` idle or `{inProgress, rootDir, folderPath, startedAt}` running. Concurrent `asset.dump_folder` calls reject with `DUMP_IN_PROGRESS` naming the active folder; registry-loading state returns `REGISTRY_LOADING` instead of blocking. State moved into `FPluginState::GetFolderDump()` (new `State/AsyncFolderDumpState.h`) with `ResetForTesting` coverage; ticker cleanup lives in `ShutdownModule` via `FCoreDelegates::OnPreExit`. Four new tests added (`StatusIdleByDefault`, `InvalidFolderPath`, `KickoffRunsAndCompletes`, `RejectsConcurrent`) using `FTSTicker::GetCoreTicker().Tick()` to drain. Live verification on `/App/App/UI`: kickoff 111 ms, sweep ~40 s, 3044 files written, concurrent-reject + status transitions confirmed.
