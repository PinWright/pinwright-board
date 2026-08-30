---
id: B-sortedjson-not-enforced-on-disk-writes
title: "Disk-writing call sites bypass SortedJsonWriter and produce non-deterministic JSON"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, determinism, goldens]
---

# Disk-writing call sites bypass SortedJsonWriter and produce non-deterministic JSON

Three on-disk JSON writers serialize directly through `FJsonSerializer::Serialize`
instead of `SortedJsonWriter::SerializeSortedJsonObject`. `FJsonObject::Values` is
a hash-ordered `TMap`, so the resulting files have unstable key order across runs.
That breaks golden comparisons, makes diffs noisy, and defeats the whole reason
`SortedJsonWriter` exists ("Stable line order across runs — diffable on disk,
round-trippable for tests" — `Utils/SortedJsonWriter.h:11`).

Confirmed sites (all write to disk, all wrap a single root `FJsonObject`):

1. `Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp:1056`
   — writes the `.dumpcache.json` sidecar via `FFileHelper::SaveStringToFile`
   immediately after the raw serialize. The cache decides aspect freshness; if it
   churns line order on every write, every diff over the asset-dump tree is
   polluted with spurious churn.

2. `Source/PinWright/Private/Handlers/Asset/AssetMetadataHandler.cpp:104`
   — serializes nested object/array metadata values that get stored as
   `UMetaData` string blobs on the asset. The blobs land in `.uasset` bytes,
   making BP/Asset diffs non-deterministic whenever metadata round-trips.

3. `Source/PinWright/Private/Handlers/Blueprint/BlueprintApiIndexHandler.cpp:207`
   — writes the merged blueprint API index JSON to `IndexPath` via
   `FFileHelper::SaveStringToFile`. Used by agents for class/function lookup;
   diff-friendly on-disk order matters for review and regeneration audits.

`Utils/SortedJsonWriter.h` already exposes the right wrapper:

```cpp
inline FString SerializeSortedJsonObject(const TSharedPtr<FJsonObject>& Obj);
```

…which uses `TPrettyJsonPrintPolicy` with stable key order. Each site is a
~10 LOC drop-in: build `Body = SortedJsonWriter::SerializeSortedJsonObject(Root)`
and remove the local `TJsonWriterFactory<>::Create` / `FJsonSerializer::Serialize`
plumbing. The condensed-policy writer at site 3 will switch to pretty; that is
the intent (goldens and human diffs both benefit).

Site 2 is slightly different: it serializes individual `TSharedPtr<FJsonValue>`
nodes (objects or arrays) rather than a root object. The wrapper covers objects
directly; arrays fall through `WriteSortedValue`, so a sibling helper
`SerializeSortedJsonValue(const TSharedPtr<FJsonValue>&)` should be added to
`SortedJsonWriter.h` to cover this case cleanly.

**Fix:**
1. Add `SortedJsonWriter::SerializeSortedJsonValue` companion in the header.
2. Replace the three raw `FJsonSerializer::Serialize` calls with the sorted
   wrapper at each site.
3. Bump the `.dumpcache.json` aspect version (per `CLAUDE.md` Aspect Version
   Bumping rule) so existing caches regenerate into deterministic form on
   first reuse.

**Enforcement (optional follow-up):** add a build-time grep guard or a small
unit test that scans `Private/Handlers/**/*.cpp` for `FJsonSerializer::Serialize`
calls feeding into `FFileHelper::SaveStringToFile` and fails if any
non-allowlisted site appears. Keeps the policy from rotting.

## History
- `#1-initial-scope` `OPEN` reporter — Filed three disk-writing call sites (AssetDumpCache.cpp:963, AssetMetadataHandler.cpp:104, BlueprintApiIndexHandler.cpp:207) that bypass SortedJsonWriter and emit hash-ordered JSON; verified each writes to disk, wrapper API `SerializeSortedJsonObject` exists in `Utils/SortedJsonWriter.h`. ~10 LOC fix per site plus a new `SerializeSortedJsonValue` helper for site 2.
- `#2-routed-three-sites-through-sorted-writer` `IN-REVIEW` developer — Added `SortedJsonWriter::SerializeSortedJsonValue` companion in `Utils/SortedJsonWriter.h` for standalone values (dispatches containers through `WriteSortedValue`; wraps/unwraps primitive roots since UE's writer rejects bare scalars). Routed all three disk writers through the sorted wrapper: `AssetDumpCache.cpp` `WriteCacheRecord` → `SerializeSortedJsonObject`; `AssetMetadataHandler.cpp` metadata-blob default case → `SerializeSortedJsonValue`; `BlueprintApiIndexHandler.cpp` API-index write → `SerializeSortedJsonObject` (condensed→pretty, intended). Swapped each orphaned writer/policy include for `Utils/SortedJsonWriter.h` (kept `JsonSerializer.h` in `BlueprintApiIndexHandler.cpp` — still used by two `Deserialize` calls). Bumped `AssetDumpCacheVersion` 1→2 in `AssetDumpCache.h` so stale caches regenerate deterministically. Regression test `Tests/Utility/TestAssetDumpCacheSortedJson.cpp` (`FAssetDumpCacheSortedJsonDeterminismTest`) writes one record via production `WriteCacheRecord`, reads the file, and asserts the eight top-level keys appear in lexical order (the `SortedJsonWriter` contract) — fails if reverted to raw hash-ordered serialize. (A sorted-line / same-run byte-equality check cannot catch the regression: TMap hash order is stable within a run, so two same-run serializations are identical regardless of the fix; the guarantee is per-run sorted key order, which is what the test asserts.)
- `#3-corrective-fix` `IN-REVIEW` developer — Entry #2 overclaimed: site 2 (`AssetMetadataHandler.cpp` `asset.set_metadata` default case) was NOT routed through the sorted writer in the working tree — grep showed 0 `SortedJson` / 1 raw `FJsonSerializer::Serialize` there (the change had been reverted by a later landscape fix-agent as out-of-scope). Sites 1 (`AssetDumpCache.cpp`) and 3 (`BlueprintApiIndexHandler.cpp`) were correctly routed. This completes the gap: replaced the raw `TJsonWriterFactory<>`/`FJsonSerializer::Serialize` default case in `AssetMetadataHandler.cpp` with `SortedJsonWriter::SerializeSortedJsonValue(Val)` and swapped the now-orphaned `Serialization/JsonSerializer.h` include for `Utils/SortedJsonWriter.h` (no other consumer in the file). The `SerializeSortedJsonValue` helper from #2 was still present in the header and reused as-is. Did not touch the `FAsyncResponseToken` migration in `asset.set_tags`. Added `FAssetMetadataSortedJsonValueTest` to `Tests/Utility/TestAssetDumpCacheSortedJson.cpp`: serializes an object value with reverse-lexical insertion-order keys (zulu/mike/alpha) via `SerializeSortedJsonValue` and asserts they emit in lexical order — fails if the default case reverts to raw hash-ordered serialize. Files: `Handlers/Asset/AssetMetadataHandler.cpp`, `Tests/Utility/TestAssetDumpCacheSortedJson.cpp`.
- `#4-verify-fix` `DONE` tester — Verified site 2 live: created temp BP `/Game/App/UI/Test/W_McpVerifyTemp_sortedjson`, called `asset.set_metadata` with `{"VerifyBlob":{"zulu":1,"mike":2,"alpha":3}}` (reverse-lexical insertion order → hits the default-case `SerializeSortedJsonValue` path), then `asset.get_metadata` returned the blob as `{"alpha": 3, "mike": 2, "zulu": 1}` — keys in strict lexical order, and pretty-printed (`\r\n\t`) confirming the `TPrettyJsonPrintPolicy` sorted wrapper rather than the old condensed raw serialize. Deleted the temp BP. Live behavior proves the corrective fix; PASS.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 3 body citations repointed in place and verified against plugin HEAD `ef8a1f1b`. Three citations, all checked. `AssetDumpCache.cpp:963` had drifted onto `MakeCacheRecord`'s opening brace and is repaired to `:1056` (the serialize; `SaveStringToFile` at `:1063`, both inside `WriteCacheRecord` at `:1031`). `AssetMetadataHandler.cpp:104` and `BlueprintApiIndexHandler.cpp:207` land exactly, no drift. **All three fixes have landed** — `:1056` and `:207` now call `SortedJsonWriter::SerializeSortedJsonObject` and `:104` `SerializeSortedJsonValue`, so the body's “raw serialize” reads as pre-fix history. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
