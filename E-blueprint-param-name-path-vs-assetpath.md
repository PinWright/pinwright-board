---
id: E-blueprint-param-name-path-vs-assetpath
title: "blueprint.* handlers split on path vs assetPath param name"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# blueprint.* handlers split on path vs assetPath param name

`blueprint.compile` requires `path` but sibling `blueprint.compile_bpir`, `blueprint.decompile`, `blueprint.inspect`, and most `blueprint.graph.*` require `assetPath`. Passing `assetPath` to `blueprint.compile` returns `MISSING_REQUIRED_PARAM: Missing required parameter 'path'`. CLAUDE.md's "camelCase and snake_case aliases" rule does not cover this — `path` and `assetPath` are distinct names, not casing variants.

The split is per-file, not random:
- `path` cluster (~25 RPCs): goes through `BlueprintHandlerUtils.cpp::ResolveBlueprintPath`, which accepts `requestedPath` / `path` / `name` / `blueprintPath` / `blueprint_path` (but **not** `assetPath`). Files: BlueprintCompileHandler, BlueprintInfoHandler, BlueprintPropertyHandler, BlueprintFunctionHandler, BlueprintEventHandler, BlueprintComponentHandler, BlueprintReparentHandler, BlueprintTypeDefinitionHandler, BlueprintVariableCleanupHandler.
- `assetPath` cluster (~25 RPCs): uses `Ctx.RequireString("assetPath", ...)` directly, no alias fallback. Files: BlueprintDecompilerHandler, BlueprintInspectHandler, BlueprintGraphHandler, BlueprintGraphOrphanHandler, BpirCompilerHandler, BpirExpressionHandler, BlueprintCodeCompilerHandler. (One handler — `blueprint.graph.list_graphs` — documents `blueprintPath` as an alias inline; the rest do not.)

Structural precedent: `E-class-name-format-inconsistency` (DONE) unified class-name parsing through a shared `ResolveUClass`. Same pattern applies here.

**Fix:** Keep the resolver changes from `#2`, but move wire-level alias acceptance into dispatcher metadata. Add alias metadata to `FParamSpec`, have `RpcDispatcher::ValidateHandlerParams` treat aliases as satisfying required params and as known params, then annotate resolver-backed Blueprint path specs (`path`, `assetPath`, `blueprintPath`) with the shared path aliases. Do not solve this by making canonical params optional per handler; that leaves aliases undiscoverable and duplicates validation.

## History
- `#1-initial-repro` `OPEN` reporter — Called `blueprint.compile` with `args:{assetPath:"/App/App/UI/W_PhotoPopup"}` → `MISSING_REQUIRED_PARAM`. Retry with `path` succeeded. Inspected source: `BlueprintCompileHandler.cpp:18` declares `RPC_PARAM_REQ("path", ...)` and resolves via `ResolveBlueprintPath` (BlueprintHandlerUtils.cpp:1242), whose FieldNames list omits `assetPath`. `BpirCompilerHandler.cpp:157`, `BlueprintDecompilerHandler.cpp:49`, `BlueprintInspectHandler.cpp:28` all declare `RPC_PARAM_REQ("assetPath", ...)`. Roughly 50/50 split across the namespace.
- `#2-add-assetpath-alias` `IN-REVIEW` developer — Added 'assetPath' to FieldNames tables of ResolveBlueprintPath and ResolveExplicitBlueprintPath in BlueprintHandlerUtils.cpp. Migrated assetPath-cluster handlers (BlueprintInspect, BlueprintDecompiler, BpirCompiler, BpirExpression, BlueprintCodeCompiler, BlueprintGraphOrphan) to call ResolveBlueprintPath. Updated BlueprintGraphHandler::ResolveBlueprintAndGraph path preamble. RPC_PARAM_REQ decls unchanged (aliases stay runtime-only, matching existing 5-alias precedent). Added unit-test coverage at Tests/Blueprint/TestResolveBlueprintPath.cpp.
- `#3-returned-dispatcher-rejects-aliases` `OPEN` tester — Returned: aliases still fail at runtime. Test: `blueprint.compile {assetPath:"/App/App/UI/W_PhotoPopup"}` → `MISSING_REQUIRED_PARAM 'path'`; `blueprint.decompile {path:"..."}` → `MISSING_REQUIRED_PARAM 'assetPath'`; `blueprint.compile {blueprintPath:"..."}` (pre-existing alias) → `MISSING_REQUIRED_PARAM 'path'`. Root cause: `RpcDispatcher.cpp::ValidateHandlerParams` (lines 26-41) strictly checks `Params->HasField(Spec.Name)` against the `RPC_PARAM_REQ` declaration BEFORE the handler body runs, so `ResolveBlueprintPath`'s FieldNames table is never reached when the canonical name is missing. The "existing 5-alias precedent" assumption in #2 is itself broken — `blueprintPath` etc. were never actually functional aliases at the wire level. Fix must either (a) teach the dispatcher to honor an alias list on `FParamSpec`, or (b) declare the canonical param as optional (`RPC_PARAM_OPT`) and validate presence inside the handler.
- `#4-dispatcher-param-aliases` `IN-REVIEW` developer — Added dispatcher-level FParamSpec aliases, wired resolver-backed Blueprint path params to shared aliases, exposed aliases in discovery/help, and added dispatcher regression coverage for alias-only required params.
- `#5-verify-fix` `DONE` tester — Verified: `blueprint.compile {assetPath:"/App/App/UI/W_PhotoPopup"}` → `{compiled:true, status:"UpToDate"}`; `blueprint.decompile {path:"/App/App/UI/W_PhotoPopup"}` → `{success:true, bpir:"..."}`; `blueprint.compile {blueprintPath:"/App/App/UI/W_PhotoPopup"}` → `{compiled:true}`. All three repro cases from `#3` now succeed; dispatcher honors alias-only required params.
