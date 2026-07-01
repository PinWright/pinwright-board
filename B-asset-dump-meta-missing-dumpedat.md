---
id: B-asset-dump-meta-missing-dumpedat
title: "asset.dump meta.json never includes the documented dumpedAt timestamp field"
status: WONTFIX
severity: Medium
category: bug
tags: [asset-dump, meta, schema]
---

# asset.dump meta.json never includes the documented dumpedAt timestamp field

The agent-facing playbook in `CLAUDE.local.md` lists `dumpedAt` as a meta.json field; no dumped meta.json contains it. Grep on `dumpedAt`/`DumpedAt` across the entire plugin source returns zero hits.

Cache freshness can only be inferred from filesystem mtime — fragile and lost on copy/move. Affects all ~30k dumped assets.

**Repro:**
1. Inspect any meta.json under `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/`.
2. Observe: `dumpedAt` is absent. Documented fields present are `assetPath`, `assetType`, `className`, `parentClass`, `packageFlags`, `pluginVersion`, `dumpSchemaVersion`.

**Fix (proposed):** `AssetDumpBuilder.cpp:36-109` (`BuildMetaJson`) emits `assetPath`, `assetType`, `className`, `parentClass`, `packageFlags`, `pluginVersion`, `dumpSchemaVersion` — no timestamp. Either drop the doc claim or add `Meta->SetStringField(TEXT("dumpedAt"), FDateTime::UtcNow().ToIso8601())`.

## History
- `#1-initial-repro` `OPEN` reporter — `dumpedAt` is documented in `CLAUDE.local.md` as a meta.json field but is absent from every meta.json across a 30k-asset sweep. Source-tree grep on `dumpedAt`/`DumpedAt` returns zero hits.
- `#2-wontfix-doc-corrected` `WONTFIX` developer — Per-dump timestamp would churn the meta.json diff for unchanged assets on every re-dump and provides no signal beyond filesystem mtime, which is the natural cache-freshness mechanism. Resolved by removing `dumpedAt` from the schema list in `CLAUDE.local.md` instead of adding the field. No code change.
