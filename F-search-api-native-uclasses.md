---
id: F-search-api-native-uclasses
title: "No keyword search for native UClasses (actor / component / widget types)"
status: DONE
severity: Medium
category: feature
tags: [search, class-discovery, system-inspect, actor, component, widget, slate]
---

# No keyword search for native UClasses (actor / component / widget types)

There is no first-class RPC for fuzzy / keyword search over the **catalog of
native (C++) UClasses** loaded into the editor — the question "which actor /
component / Slate widget classes exist and match keyword X" has no typed
handler. The current surface answers adjacent but different questions:

- `blueprint.graph.list_node_types` — enumerates K2Node container classes for
  a Blueprint graph. Not class search.
- `blueprint.search_api` + `blueprint.build_api_index` — keyword search over
  Blueprint-callable **UFunctions**, not classes. Useful for `CallFunction`
  targets, not for "find me a component class to add".
- `asset.search` / `asset.search_assets` — asset-registry search for `/Game`
  content, including BP-derived classes via `parentClassPath`. Does **not**
  cover native classes (they live in `/Script/<Module>.<Name>`, not in the
  asset registry).
- `system.inspect.inspect_class` — exact-name resolver. Errors
  `CLASS_NOT_FOUND` unless caller already knows the precise name/path. Not
  fuzzy and returns a single class.
- `system.inspect.list_objects`, `system.inspect.find_by_class`,
  `actor.find_by_*` — operate on placed **instances** in the world, not on
  the class catalog.

Current workaround is `python.execute` to walk
`unreal.find_class` / `TObjectIterator<UClass>` via reflection, or grep
engine source. Both bypass the typed-RPC surface and the
build-index-then-search pattern that already works for BP functions.

**Use cases blocked:**

1. "What Component classes can I add to this actor?" — caller wants a ranked
   list of `UActorComponent` subclasses matching a keyword (`movement`,
   `audio`, `physics`). Today: no way without Python.
2. "What Slate / UMG widget classes exist?" — for `widget.import_xml` or
   `widget.add` authoring, caller needs to know that `UTextBlock`,
   `UButton`, `UCanvasPanel`, `UScrollBox`, etc. exist. The XML schema is
   the implicit catalog today; no RPC enumerates it.
3. "What Actor subclasses exist matching `Character` or `Light` or
   `Trigger`?" — for `actor.spawn`, caller needs to discover spawnable
   native classes without already knowing the name.

**Proposal:** Add `system.inspect.search_classes` (or
`system.inspect.list_classes` for unranked enumeration) modeled on the
existing `blueprint.search_api` pattern:

```
system.inspect.search_classes(
    query: string,               // keyword(s); ranked match on class name + module
    parentClass?: string,        // restrict to subclasses of this UClass (e.g. "ActorComponent", "UserWidget", "Actor")
    moduleFilter?: string[],     // restrict to specific modules (e.g. ["Engine", "UMG", "SlateCore"])
    includeAbstract?: bool,      // default false — exclude `CLASS_Abstract`
    includeDeprecated?: bool,    // default false — exclude `CLASS_Deprecated`
    limit?: number               // default 20
) -> {
    results: [{
        className: "ButtonComponent",      // GetName(), no U/A prefix
        fullPath: "/Script/Engine.Button", // class path
        parentClass: "Widget",
        module: "UMG",
        isAbstract: false,
        flags: ["BlueprintType", "Blueprintable"],   // selected UClass flags
        score: number
    }],
    totalMatches: number
}
```

Implementation surface: iterate `TObjectIterator<UClass>` once into an
in-process cache (rebuildable on demand or via a `build_class_index`
sibling, mirroring `blueprint.build_api_index`), apply
`parentClass` / `moduleFilter` predicates, then BM25-style keyword rank on
`GetName()` plus `GetMetaData("DisplayName")` and `GetMetaData("Category")`.
Cache invalidation only on hot-reload / module load events; for a 5.6 editor
the class count is bounded (~30k UClasses) and the index sits comfortably
in memory.

**Optional companions** (lower priority, can be follow-ups):

- `system.inspect.list_module_classes(module: string)` — every UClass owned
  by a module, useful for "what does the `Niagara` module expose".
- `system.inspect.search_structs` / `search_enums` — same shape but for
  `UScriptStruct` / `UEnum`. The BP function index already implicitly
  surfaces struct/enum names in parameter signatures, but a direct typed
  search would help when authoring `MakeStruct` / variable definitions.

**Why this matters for agent workflows:** the BPIR + `search_api` chain
already proves the pattern for function discovery. Replicating it for
classes closes the loop on "I want to add component X but don't know its
exact class name" — currently the only path is asking the user or running
Python, both of which break unattended automation.

## History
- `#1-no-native-class-search` `OPEN` reporter — Surveying discovery RPCs revealed `blueprint.graph.list_node_types` (K2Node catalog) and `blueprint.search_api` (UFunction catalog) but no peer for UClass catalog. `asset.search` only covers `/Game` BP-derived classes; `system.inspect.inspect_class` is exact-name only. No way to answer "list / search native Actor / ActorComponent / UserWidget / Slate widget classes by keyword" without `python.execute`. Proposes `system.inspect.search_classes` with `parentClass` + `moduleFilter` and an optional `build_class_index` mirroring the existing `blueprint.build_api_index` → `blueprint.search_api` pattern.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified the gap from source: `BlueprintApiIndexHandler.cpp` iterates `TObjectIterator<UClass>` but emits only class→function mappings, discards classes with zero `FUNC_BlueprintCallable` functions (line 137), and `blueprint.search_api` scores by function name, not by class name — so it's unusable as a class-catalog substitute. Confirmed no `system.inspect.search_classes` / `list_classes` / `build_class_index` handler exists (grep across `Source/`). Checked sibling tickets: `B-inspect-class-short-name-fails` (DONE) only added short-name resolution to exact-match `inspect_class`, doesn't add search; `F-asset-search-native-subclass` (DONE) covers BP-asset filtering by native parent, but for `/Game` BPs only, not the native-class catalog. The proposal sits alongside an existing coherent `F-search-api-*` series (console-commands, niagara-modules, anim-graph-nodes, etc.) and follows the same shape. Severity Medium is appropriate: a `python.execute` fallback exists (`unreal.find_class` / TObjectIterator walk), so this is worse-DX rather than blocking — High is reserved for cases with no fallback (e.g. `F-search-api-console-commands`). Title and scope are precise; the optional companions (`list_module_classes`, `search_structs`, `search_enums`) are correctly framed as follow-ups.
- `#3-implemented-search-classes` `IN-REVIEW` developer — Added `system.inspect.search_classes` at `Private/Handlers/System/ClassSearchHandler.cpp`. Lazy in-memory class catalog (`TObjectIterator<UClass>`), invalidated on `FModuleManager::OnModulesChanged`. parentClass via `ResolveUClass`; module extraction by stripping `/Script/` from outer package name; case-insensitive token-weighted substring score (class name +3, DisplayName +2, Category +1, module +1). Regression tests under `Tests/Private/World/TestClassSearch.cpp`.
- `#4-verify-search-classes` `DONE` tester — Verified: `system.inspect.search_classes` wiki exposes `query`, `parentClass`, `moduleFilter`, `includeAbstract`, `includeDeprecated`, and `limit`; live call `system.inspect.search_classes` with `{"query":"movement","parentClass":"ActorComponent","moduleFilter":["Engine"],"limit":5}` returned Engine native component classes including `CharacterMovementComponent`, `FloatingPawnMovement`, `ProjectileMovementComponent`, and `RotatingMovementComponent`, with native `/Script/Engine.*` paths and `totalMatches: 6`.
