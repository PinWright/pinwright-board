---
id: E-asset-dump-meta-sidecars-emitted-field
title: "asset.dump meta.json should declare emitted sidecars so consumers can distinguish 'fully handled' from 'unhandled class'"
status: DONE
severity: Low
category: ergonomic
tags: [asset-dump, meta-json, schema-parity, discoverability]
---

# asset.dump meta.json should declare emitted sidecars so consumers can distinguish 'fully handled' from 'unhandled class'

When `AssetDumpHandler.cpp` has no per-type writer for a given class, the
output folder contains exactly `meta.json` + `properties.json` — which is
also the legitimate steady-state for asset kinds whose structural data
genuinely fits in `properties.json` alone (simple `UDataAsset` subclasses,
`UPrimaryDataAsset` configs, `UDeveloperSettings`, etc.). Consumers reading
the dump cache cannot tell these apart:

- "This class has no type-specific sidecar because nothing more is needed" vs.
- "This class has a structural surface the dumper doesn't know how to flatten,
  and the consumer is missing data that's silently hiding in escaped struct
  strings inside `properties.json`."

This is the same class of ergonomic problem `B-asset-dump-empty-properties-no-success-marker`
and `E-asset-dump-properties-status-parity-non-bp` solve for the
`properties.json` empty-vs-error case, but for the **sidecar dispatch**
layer rather than the properties layer.

**Recommendation:** Emit one of the following on every `meta.json`:

1. **`sidecarsEmitted: ["meta.json","properties.json","static_mesh.json",...]`** — explicit list of every file the dispatcher wrote for this asset. Consumers can intersect with the canonical sidecar registry to detect missing handlers per class.
2. **`unsupportedClass: true`** on assets that fell through every per-type branch in the dispatcher — explicit "no writer is registered for this class" signal.

Option 1 is more general (covers the unhandled-class case via `properties.json` being the only structural file present, but also surfaces partial-handler bugs where one of N expected sidecars is missing). Option 2 is cheaper but only catches the binary unhandled-vs-handled split, not partial coverage.

**Why this matters:** The board already carries one ticket per missing
per-type sidecar (`B-asset-dump-non-texture2d-no-native-summary`,
`B-asset-dump-skeletal-mesh-summary-missing`,
`B-asset-dump-missing-type-sidecars-for-anim-physics-skeleton`, etc.). Each
of those was found by manually grepping the dump cache for `meta.json` +
`properties.json`-only folders and cross-referencing class names with the
dispatcher table. A `sidecarsEmitted` field would let a single sweep
auto-list every class for which no type writer exists, so future gaps
surface as data instead of needing a human to notice them.

**Fix (proposed):** In `Utils/AssetDumpBuilder.cpp::BuildMetaJson` (called
after `AssetDumpHandler` finalizes the `Files` list), serialize the
emitted-file names into `meta.sidecarsEmitted`. Bump `dumpSchemaVersion`.
Add a regression test asserting the field is present and contains the
expected set for at least one StaticMesh, one Texture2D, and one
"unhandled" class (e.g. any `UPrimaryDataAsset` subclass that has no
dedicated builder).

## History
- `#1-initial-report` `OPEN` reporter — `asset.dump` output for a class with no type-specific writer is indistinguishable from output for a class whose structural data legitimately fits in `properties.json` alone — both produce only `meta.json` + `properties.json`. Recommend emitting `meta.sidecarsEmitted: [...]` on every dump so consumers can filter "fully handled" vs "unhandled class" without maintaining their own out-of-band class taxonomy. Companion to `B-asset-dump-missing-type-sidecars-for-anim-physics-skeleton`, which is the latest in the running stream of per-class missing-sidecar tickets that this field would have let us discover programmatically.
- `#2-sidecars-emitted-attached` `IN-REVIEW` developer — Added `AttachSidecarsEmittedToMeta` helper inside the anonymous namespace of `Handlers/Asset/AssetDumpHandler.cpp`; it sorts every dispatcher-emitted `FDumpFile.Name` alphabetically and writes the array to `meta.json` as `sidecarsEmitted`. `BuildAllFilesForAsset` calls it from both the redirector short-circuit and the post-dispatch tail before `AddMetaFile()`. Skip stubs in `TickFolderDump` write meta directly without invoking the dispatcher and therefore intentionally omit the field — the existing `skipped: true` flag remains the primary signal. Bundled into the schema-v6 bump in `Utils/AssetDumpBuilder.cpp` and documented in `docs/wiki/asset.md` alongside `kind` / `blueprintType` / `compileStateAvailable` / `propertiesStatus` parity additions.
- `#3-skip-editor-offline` `SKIP` tester — Editor offline (port 19880 connection refused), cannot run live `asset.dump` to observe the new `sidecarsEmitted` field. Cache under `.editor-automation/asset-dumps/` is stale at schema v3 (predates v6 bump). Static review confirms `AttachSidecarsEmittedToMeta` is wired into both call sites in `AssetDumpHandler.cpp` (line 455 redirector short-circuit, line 808 post-dispatch tail), invoked after `Files` is populated but before `AddMetaFile()` inserts `meta.json` so the array won't self-reference. `dumpSchemaVersion` bumped to 6 in `AssetDumpBuilder.cpp:146` with matching changelog comment. No regression test for the field exists under `Private/Tests/` — body's recommendation for one is unmet, but the developer's IN-REVIEW message did not claim one.
- `#4-verify-live-dump` `DONE` tester — Verified live: `asset.dump` on `/Engine/VREditor/TransformGizmo/SM_Sequencer_Node` produced `meta.json` with `dumpSchemaVersion: 6` and `sidecarsEmitted: ["properties.json","static_mesh.json"]`; second sample `/Engine/EngineMaterials/T_Default_Material_Grid_M` (Texture2D) emitted `sidecarsEmitted: ["properties.json","texture.json"]`. Both lists are sorted, exclude the self-referencing `meta.json`, and match the dispatcher's per-type writer set for those classes — confirming the IN-REVIEW contract end-to-end.
