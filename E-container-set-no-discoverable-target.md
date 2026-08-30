---
id: E-container-set-no-discoverable-target
title: "container.set wiki names no concrete TSet target; finding one requires grepping engine source"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [container, set, docs, discoverability, property]
---

# `container.set` gives no way to find a `TSet` UPROPERTY to operate on

The `container.set` verbs (`add`/`contains`/`remove`/`clear`) all require an
`objectPath` + `propertyName` pointing at a reflected `TSet` UPROPERTY, but
**nothing in the toolset or docs tells the caller where such a property lives**.
`TSet` UPROPERTYs are rare on placed level actors, so a user with the simple
intent "drive the set lifecycle on a TSet property" first has to *hunt for a
target* before they can issue a single `container.set` call.

The 3-line `container.set` wiki overlay
(`docs/wiki-src/container.set.md`) describes the dedupe/idempotency semantics
but names **no concrete example target** and gives **no recipe** for locating a
`TSet` property (e.g. "scan a CDO with `property.list`, look for
`cppType` starting `TSet`" or a known always-present target such as
`/Script/Engine.Default__AssetManagerSettings.MetaDataTagsForAssetRegistry`).

## Evidence (this task, `container.set` namespace)

The friction note records the discovery cost verbatim: *"Struggled to locate a
TSet target: property.list on placed actors (WorldSettings, Landscape,
BP_DemoDisplay) exposed only TArray/scalar props, no TSet, so I grepped the
installed engine source for editable TSet UPROPERTYs and landed on
AssetManagerSettings.MetaDataTagsForAssetRegistry — finding a TSet at all was
non-trivial (placed-actor TSet UPROPERTYs are rare)."*

Call-log shape before the first `container.set` call could run (pure discovery
overhead, all `ok:true` — no functional failure, just steps):

- 1× root-namespace wiki-nav + reads of the `container.set` + per-verb wiki pages
- `actor.list` (enumerate level actors)
- `property.list` ×4 — `WorldSettings_1`, `Landscape_0`, `BP_DemoDisplay_C_0`
  (all TArray/scalar, **no TSet**), then `Default__AssetManagerSettings`
  (found `MetaDataTagsForAssetRegistry cppType=TSet`)
- **out-of-band**: grep of installed UE engine source for an editable `TSet`
  UPROPERTY — a step the MCP cannot observe and a non-MCP user may not be able
  to perform

That is ~7 discovery calls plus an engine-source grep to satisfy a one-line
intent. None of it is a bug — it is missing discoverability/docs.

## What it should do

- The `container.set` wiki overlay should **name at least one concrete,
  always-present `TSet` target** (the engine CDO
  `/Script/Engine.Default__AssetManagerSettings.MetaDataTagsForAssetRegistry`
  is a `TSet<FName>` present in every project) so a caller can copy-paste a
  working `objectPath`+`propertyName` instead of hunting.
- It should give the **discovery recipe in-product**: `property.list {objectPath,
  includeAll:true}` then look for entries whose `cppType` begins `TSet`. The
  now-shipped `property.list` `nameMatch` filter
  (`F-property-list-name-filter`, DONE) narrows by *name* but not by
  *container kind* — note that limitation so callers don't expect a
  `cppType`/container filter that doesn't exist.
- (Optional, larger) a container-kind filter on `property.list` (`cppTypePrefix`
  / `containerKind:"set"`) would let a caller enumerate all `TSet` props on an
  object in one call instead of scanning the full listing — but the docs fix
  above resolves the immediate friction without new code.

**Fix:** Downstream wiki edit to `docs/wiki-src/container.set.md` (and likewise
`container.map.md` / `container.array.md` if they share the gap): add a "finding
a target" line naming the AssetManagerSettings CDO target and the
`property.list` → `cppType` scan recipe. Docs-only; no handler change required.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of a `container.set`
  lifecycle task (outcome was a separately-filed functional bug,
  `B-container-set-fname-lookup-broken`; this ticket is the distinct
  DISCOVERABILITY angle). Before any `container.set` call could run, the session
  spent root + per-verb wiki reads, `actor.list`, and `property.list` ×4 across 3
  placed actors (all TArray/scalar, no TSet) and finally **grepped installed
  engine source** to find `AssetManagerSettings.MetaDataTagsForAssetRegistry`
  (`TSet<FName>`). The 3-line `container.set` overlay names no example target and
  no `cppType` discovery recipe; `property.list`'s `nameMatch` filter narrows by
  name only, not container kind. ~7 discovery calls + an out-of-band grep to
  satisfy a one-line intent.
- `#2-fix` `IN-REVIEW` developer — Docs-only fix (no handler change). Added a
  `## Finding a <T> target` namespace section to each of the three container
  overlays: `Docs/wiki-src/container.set.md` (names the always-present engine CDO
  `/Script/Engine.Default__AssetManagerSettings.MetaDataTagsForAssetRegistry`,
  a `TSet<FName>`, as a copy-paste `objectPath`+`propertyName`),
  `Docs/wiki-src/container.map.md` (CDO
  `/Script/Engine.Default__UserInterfaceSettings.HardwareCursors`, a reflected
  `TMap`), and `Docs/wiki-src/container.array.md` (CDO
  `/Script/Engine.Default__AssetManagerSettings.DirectoriesToExclude`, a
  `TArray<FDirectoryPath>` — all three verified against UE 5.7 engine source).
  Each section also gives the in-product discovery recipe — `property.list
  {objectPath, includeAll:true}` then scan for an entry whose `cppType` begins
  `TSet`/`TMap`/`TArray` — and notes `property.list` has no `cppType`/container-kind
  filter (only the `nameMatch` name substring filter), so callers scan the listing
  themselves. These sections sit in the overlay prelude (before any `### ` line) so
  they render on the namespace page but are cut from the root index (per the
  structural quality rules), keeping the root lean. Regression test:
  `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestContainerTargetDiscoveryDocs.cpp`
  (`FContainerSetTargetDiscoveryDocTest` / `FContainerMapTargetDiscoveryDocTest` /
  `FContainerArrayTargetDiscoveryDocTest`) renders each namespace page through the
  live `WikiHandler::RenderPage` path (same entry the gateway uses) and asserts the
  overlay-exclusive markers (CDO objectPath, property name, the `cppType`+`property.list`
  recipe, and the `nameMatch`/`container-kind filter` limitation note) survive —
  reverting the overlay sections fails the test.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
