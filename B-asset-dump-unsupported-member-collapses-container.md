---
id: B-asset-dump-unsupported-member-collapses-container
title: "One undecodable member collapsed its whole container into a single `{_kind: unsupported}` marker — FSoftObjectPath lost its AssetPath, FNavAgentProperties / FAssetBundleData / TMap<FAnimationAttributeIdentifier,...> dumped as a bare marker"
status: IN-REVIEW
severity: High
category: bug
tags: [asset-dump, properties, property-export, unsupported-marker, struct-expansion, data-loss, regression]
encounters: 1
lastSeen: 2026-09-09T10:00:00Z
---

# An unsupported leaf erased every sibling that was decodable

`ExportPropertyToJsonValue` stamped a single `_kind: unsupported` marker over the entire
enclosing struct, array, map or set the moment one member could not be decomposed. The
sibling members were never emitted, so the dump reported "this type is not supported"
about types that are supported and whose values were sitting right there.

Observed in `asset-dumps/` on `X:\src\unreal\unreal-fpv-dev`:

- `FSoftObjectPath` — the inner `SubPathString` (`FUtf8String`) is the undecodable member,
  so the whole struct collapsed and the **`AssetPath` was lost**. Every soft reference in
  the mirror read as an opaque marker.
- `FNavAgentProperties`, `FAssetBundleData`, `TMap<FAnimationAttributeIdentifier, ...>` —
  emitted as a bare `{"_kind":"unsupported"}` object with no content at all.

This is the `High` band by the README rubric — not a crash, but silent wrong data on the
normal path: a consumer reading the mirror concludes the property has no readable value.

Related but not the same defect: `B-asset-dump-property-unsupported-sentinel-on-common-types`
(DONE) was about **leaf types the exporter did not handle** (uint32/int8/TWeakObjectPtr/…)
and was fixed by widening leaf coverage. This one is about the **recursion policy** — a
handled container abandoning its decodable members because of one unhandled leaf — and is
fixed in a different function with a different mechanism.

**Fix (shipped, plugin `fa755a4f`):** recursion now runs through
`ExportNestedPropertyToJsonValue`
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\PropertyExport.cpp:288`),
which marks the offending member **in place** and keeps every sibling it can decompose.
Marker tolerance is an explicit opt-in on the export source
(`FPropertyExportSource::WithUnsupportedMarkers()` / `AllowsUnsupportedMarkers()`,
`Private\Utils\PropertyExport.h:52-110`), **off by default**, so only the loss-tolerant
dump path takes it; the strict RPC-response path still rejects the whole shape when a
member is undecodable, which is the correct behaviour for a response a caller will parse
by schema. Aspect versions bumped `properties.json 7->8`, `scs.json 4->5`, and dump core
`2->3` because the shared exporter changed for every UPROPERTY-serializing aspect.

## History
- `#1-container-collapse-observed` `OPEN` reporter — Found while reviewing the committed dump mirror on `X:\src\unreal\unreal-fpv-dev` (UE 5.8): `FSoftObjectPath` values throughout `asset-dumps/App` carried `{"_kind":"unsupported"}` instead of their `AssetPath`, and `FNavAgentProperties`, `FAssetBundleData` and `TMap<FAnimationAttributeIdentifier, ...>` did the same. Traced to `ExportPropertyToJsonValue` applying the unsupported marker at container level rather than at the member that could not be decomposed, so one undecodable leaf (here `SubPathString`, an `FUtf8String`) erased the whole struct. Distinct from `B-asset-dump-property-unsupported-sentinel-on-common-types` (DONE, leaf-type coverage) — same marker text, different code path and different fix.
- `#2-nested-export-keeps-siblings` `IN-REVIEW` developer — Fixed in plugin commit `fa755a4f` ("Fix three asset-dump regressions in property and bpir export"). Recursion moved to `ExportNestedPropertyToJsonValue` (`PropertyExport.cpp:288`), which marks the undecodable member in place; tolerance is an explicit flag on `FPropertyExportSource` (`WithUnsupportedMarkers()`, `PropertyExport.h:52-110`), default off, so the strict RPC-response path is unchanged and still rejects the whole response shape on an unsupported member. Aspect versions `properties.json 7->8`, `scs.json 4->5`, dump core `2->3`. **Verified** by re-sweeping `asset-dumps/App` with this build: 8,388 / 8,388 assets, 0 skips, **no** `FSoftObjectPath`, `FNavAgentProperties` or `FAssetBundleData` markers remain, and the 24 remaining `FUtf8String` markers are inner `SubPathString` members whose parent `FSoftObjectPath` now carries a real `AssetPath` — i.e. exactly the in-place marking the fix intends. Awaiting an independent tester: the strict-path claim (RPC responses still reject rather than tolerate) was not exercised in this verification.
