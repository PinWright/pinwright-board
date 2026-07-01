---
id: B-properties-error-status-empty-payload-ambiguous
title: "Error-status assets emit empty properties.json (4 bytes) without distinguishing failure signal"
status: DONE
severity: Medium
category: bug
tags: [properties, error-handling, schema]
---

# Error-status assets emit empty properties.json

When `meta.json` records `propertiesStatus.status = "error"` with `reason = "generated_class_missing"`, the corresponding `properties.json` exists but is 4 bytes (`{}\n`). Distinct from [B-asset-dump-empty-properties-no-success-marker](B-asset-dump-empty-properties-no-success-marker.md) (which is about successful dumps that legitimately have no properties — needing a success sentinel).

This ticket: failed dumps leave a sidecar that looks like a successful empty dump.

## Sample

- `App/App/LevelBlueprints/B_SearchMode_Mine` — properties.json is 4 bytes, meta.json says `status: error, reason: generated_class_missing`
- `App/App/LevelBlueprints/B_GhostDrone` — same

## Fix sketch

In `AssetDumpHandler.cpp`, when `propertiesStatus.status == "error"`:
1. Don't write properties.json at all (and drop it from `sidecarsEmitted`), OR
2. Write a sidecar with a clear failure marker: `{"_dumpFailed": true, "reason": "generated_class_missing"}`

Option 1 is cleaner — consumers check meta.json's `propertiesStatus` for the truth.

## History
- `#2-error-sidecar-omitted` `IN-REVIEW` implementer — null `GeneratedClass` now omits `properties.json` while preserving `meta.json.propertiesStatus = { status: "error", reason: "generated_class_missing" }`; tests updated for absent sidecar and explicit status.
- `#1-error-status-empty-payload` `OPEN` reporter — distinct from B-asset-dump-empty-properties-no-success-marker. Error case needs different handling than legitimately-empty case.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/App/LevelBlueprints/B_SearchMode_Mine` returned `writtenPaths` = [meta.json, bpir.txt] and `skipped` = [{path: "properties.json", reason: "Blueprint has no GeneratedClass; CDO unavailable."}]. On-disk dir contains only meta.json + bpir.txt (no properties.json). meta.json.sidecarsEmitted = ["bpir.txt"]; propertiesStatus preserved as {status: "error", reason: "generated_class_missing"}.
