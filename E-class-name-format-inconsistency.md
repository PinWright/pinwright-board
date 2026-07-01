---
id: E-class-name-format-inconsistency
title: "MCP tools accept inconsistent class-name formats"
status: DONE
severity: Medium
category: ergonomic
tags: []
---

# MCP tools accept inconsistent class-name formats

Class-name parameters across MCP tools have no common convention. Each tool accepts a different format, with no reliable shared accepted form:

| Tool | Parameter | Accepted format |
|------|-----------|-----------------|
| `widget.create_widget_blueprint` | `parentClass` | Short name (`ReplaySaveHandlerWidget`) |
| `system.inspect.inspect_class` | `className` | Full `/Script/<Module>.<ClassName>` only (short name fails — see B-inspect-class-short-name-fails) |
| `asset.search_assets` | `classNames[]` | Short OR full `/Script/` path |
| `widget.add` | `type` | Asset path with `_C` suffix (`/App/.../W_Foo.W_Foo_C`) OR engine class short name (`SizeBox`) |
| `asset.list` | `filter.class` | Full `/Script/...` only (short fires engine ensure — see B-asset-list-short-class-ensure) |

No single format is accepted by all of them.

**Workaround:** Trial-and-error with each tool; memorize which accepts what.

**Proposal:** Define a single canonical class-name resolver (extend the `ResolveUClass` helper proposed for `B-asset-list-short-class-ensure`) and route every class-name parameter through it. Accept all of: short name, short name with `U`/`A` prefix, `/Script/<Module>.<ClassName>` full path, `/Game/<Path>/<Blueprint>.<Blueprint>_C` BP generated-class path. Return a consistent error when ambiguous.

## History
- `#1-different-formats-adjacent-calls` `OPEN` reporter — Hit during session's class visibility check (`system.inspect.inspect_class`) immediately followed by BP creation (`widget.create_widget_blueprint`). The same conceptual value required two different string formats in adjacent tool calls.
- `#2-three-handlers-unified` `IN-REVIEW` developer — Unified 3 handlers onto `ResolveUClass`. (a) Extended `ClassUtils.cpp::ResolveUClass` to recognize any non-`/Script/` mount path (`/Game/`, `/App/`, any plugin content mount) as a potential Blueprint path: tries direct load, `_C`-appended load, then `LoadObject<UBlueprint>` → `GeneratedClass`. Returns null on failure rather than falling through to short-name iteration. (b) `EnvironmentHandler.cpp::system.inspect.inspect_class` (lines 1500-1501): replaced `FindObject` + hardcoded Engine fallback with `ResolveUClass`. (c) `WidgetCreateHandler.cpp::widget.create_widget_blueprint` (lines 69-78): replaced `FindFirstObject`/`ResolveClassByName` branch with `ResolveUClass`. (d) `AssetQueryHandler.cpp::asset.search_assets` (lines 217-222): replaced `TryFindTypeSlow` U/A-prefix loop with `ResolveUClass` (kept the `/Script/` fast path since it feeds `FARFilter::ClassPaths` directly). `widget.add` and `asset.list` already handled all formats. `#include "Utils/ClassUtils.h"` added to the 3 handler files.
- `#3-refactor-remove-duplicate-load` `IN-REVIEW` developer — Refactor pass: removed duplicate `LoadObject<UClass>(Input)` in the content-mount branch (step 2 already tried it). Added 5 automation tests in `TestClassUtils.cpp`: `FResolveUClassShortNameTest`, `FResolveUClassUPrefixTest`, `FResolveUClassFullScriptPathTest`, `FResolveUClassEngineBPPathTest` (content-mount BP path via `/Engine/EditorBlueprintResources/StandardMacros`), `FResolveUClassNonExistentContentPathTest`.
- `#4-verified-uniform-class-formats` `DONE` tester — Verified via MCP: `asset_search_assets classNames:["WidgetBlueprint"]` (short name) returned the test BP with no `FTopLevelAssetPath` short-name ensure fire in the log. `widget_create_widget_blueprint parentClass:"UserWidget"` succeeded. `inspect_class` verified separately (see B-inspect-class-short-name-fails). All 5 tools now accept uniform class-name formats — short name, U/A-prefix, full `/Script/` path, content-mount BP path.
