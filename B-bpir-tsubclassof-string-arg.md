---
id: B-bpir-tsubclassof-string-arg
title: "`compile_bpir` can't resolve TSubclassOf pin from full class path string"
status: DONE
severity: High
category: bug
tags: []
---

# `compile_bpir` can't resolve TSubclassOf pin from full class path string

BPIR `call` instructions passing a TSubclassOf argument via full class path string fail: `COMPILE_FAILED: Could not resolve value '<path>' for pin '<PinName>'`. Same path works in `widget.create_widget_blueprint parentClass` and `widget.add type`, but `compile_bpir` rejects all three forms (`_C` suffix, no suffix, short name) for a TSubclassOf function parameter.

**Repro:**
```
entry custom_event TestPushLayer() {
    %owner = call GetOwningPlayer()
    %w = call PushContentToLayerForPlayer(OwningPlayer: %owner, WidgetClass: /App/App/UI/W_Error.W_Error_C, LayerName: (TagName="UI.Layer.Menu"))
}
```
→ `COMPILE_FAILED: Line 3: Could not resolve value '/App/App/UI/W_Error.W_Error_C' for pin 'WidgetClass'`

**Impact:** Blocks BPIR patterns taking a widget or actor class as argument: `PushContentToLayer`, `CreateWidget`, `SpawnActor`, etc.

**Workaround:** Declare a class variable on the BP, set default via `blueprint_set_default`, reference as `$ClassVar` in BPIR.

**Proposal:** Support `class<T>` literal syntax (similar to existing `cast<T>` / `make<T>`) for TSubclassOf pin values. Reuse the path-to-class resolver already added for FClassProperty in the `B-softclassptr-silent-fail` fix.

## History
- `#1-initial-repro` `OPEN` reporter — Hit while probing `W_Error` push path from `W_ReplaySaveHandler`. Three path variants all failed.
- `#2-resolve-uclass-wire` `IN-REVIEW` developer — Replaced `FindFirstObjectSafe<UClass>` + C++-prefix-strip retry in BpirCompiler.cpp `WireArguments` class-resolution fallback (~lines 4174–4221) with a single `ResolveUClass(Arg.Value)` call from `Utils/ClassUtils.h`. BPIR TSubclassOf pins now accept the same formats as `widget.create_widget_blueprint` and `asset.search_assets`: short name, `/Script/Module.Class`, `/Game/.../BP`, `/App/.../BP.BP_C`. Added `#include "Utils/ClassUtils.h"` to BpirCompiler.cpp.
- `#3-verified-class-path` `DONE` tester — Verified via live MCP on `W_McpVerifyTemp`: `call CreateWidget(OwningPlayer: %p.ReturnValue, Class: /App/App/UI/W_PhotoPopup.W_PhotoPopup_C)` compiles clean; pin inspection shows `pinType: "class", defaultObjectPath: "/App/App/UI/W_PhotoPopup.W_PhotoPopup_C"`. Regression test added: `FCompilerIntegrationTSubclassOfClassPathTest` in `TestCompilerDispatchersOps.cpp` (uses `/Script/UMG.UserWidget` for test-context portability).
