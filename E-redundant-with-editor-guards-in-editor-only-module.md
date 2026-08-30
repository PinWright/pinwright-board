---
id: E-redundant-with-editor-guards-in-editor-only-module
title: "Remove redundant editor guards and dead local/version defines"
status: DONE
severity: Low
category: ergonomic
tags: [cleanup, defines, tests]
---

# Remove redundant editor guards and dead local/version defines

The `EditorAutomationRpcGateway` module is declared `Type: Editor` in the `.uplugin`. Every source file in it is only compiled in the editor module context, but many files still wrap bodies in `#if WITH_EDITOR ... #endif`, carry unreachable `!WITH_EDITOR` fallbacks, or use `WITH_EDITORONLY_DATA` as a marker. The plugin's supported range is UE 5.4+, so local feature defines and version gates that only preserve pre-5.4 code paths are also dead conditionals.

Examples:
- `Source/PinWright/Private/Tests/Blueprint/TestValidateBlueprintGraphIntegrity.cpp` — outer `#if WITH_EDITOR && MCP_TEST_HAS_CREATE_DELEGATE`.
- `Source/PinWright/Private/Tests/Blueprint/TestCascadeCreateDelegateCleanup.cpp` — same.
- `Source/PinWright/Private/Tests/Blueprint/TestValidateStaleCreateDelegateGuid.cpp` — same with `MCP_TEST_HAS_CREATE_DELEGATE_GUID`.
- `Source/PinWright/Private/Tests/Bpir/TestBpirCreateDelegateImplicitSelf.cpp` — same.
- `Source/PinWright/Private/Tests/Bpir/TestBpirPhase0CascadesCreateDelegates.cpp` — same.
- `Source/PinWright/Private/Tests/Utility/TestAssetDumpInstancedSubobjects.cpp` and the other three `TestPropertyUtils*.cpp` files — `#if WITH_EDITOR` wrapping the entire test body and the paired UCLASS header include.
- `Source/PinWright/Private/Handlers/Audio/MetaSound/*` and paired tests — hardcoded UE 5.4+ MetaSound builder/frontend defines that select obsolete fallback branches.
- `Source/PinWright/Private/Handlers/Geometry/*` — hardcoded Geometry Script support defines that keep unsupported branches alive.
- `Source/PinWright/Private/Handlers/AI/*` — hardcoded StateTree, Smart Objects, Mass AI, EnvQuery, and Behavior Tree graph defines where real header probes or editor-only code should remain.
- Many other test cpps and handlers gate code with `#if WITH_EDITOR`, `!WITH_EDITOR`, or `WITH_EDITORONLY_DATA`.

**Impact:** none functional. Cleanup-only. Removing the guards and dead defines simplifies the code and removes conditional compilation layers that confuse tooling (IWYU first-include checks were complaining about `.h` files placed inside a `#if WITH_EDITOR` block in `Tests/Utility/`).

**Fix:** sweep `#if WITH_EDITOR` / `#endif // WITH_EDITOR`, unreachable `!WITH_EDITOR` arms, and redundant `WITH_EDITORONLY_DATA` guards out of `.cpp` and `.h` files under `Source/PinWright/`. Keep real optional-module, `__has_include`, platform, UE 5.5+ / 5.6+ / 5.7, and `WITH_DEV_AUTOMATION_TESTS` gates. Delete hardcoded local `MCP_*` feature defines and UE 5.4+ version gates where the supported body is now unconditional.

## History
- `#1-initial-report` `OPEN` reporter — Filed while cleaning up Unity ODR collisions after merging the standalone tests module into the main editor-only module. The `WITH_EDITOR` guards are unreachable dead code in this module.
- `#2-scope-broadened` `OPEN` reporter — Broadened from `WITH_EDITOR`-only cleanup to redundant editor guards plus dead local/version defines after confirming the module is editor-only and the supported range is UE 5.4+.
- `#3-editor-guard-define-cleanup` `IN-REVIEW` developer — Removed redundant `WITH_EDITOR`, `!WITH_EDITOR`, and `WITH_EDITORONLY_DATA` guards from `Source/EditorAutomationRpcGateway`; stripped dead hardcoded local feature defines and UE 5.4+ version gates; preserved real optional-module, platform, 5.5+/5.6+/5.7, and `WITH_DEV_AUTOMATION_TESTS` gates. Verified with static `rg` sweeps and `git diff --check`; no build or tests run.
- `#4-returned-guards-remain` `OPEN` tester — Returned: redundant editor-only guards still remain under `Source/EditorAutomationRpcGateway`, including `#if WITH_EDITOR` / `#if !WITH_EDITOR` in `Private/Handlers/AI/AIHandler.cpp`, `WITH_EDITORONLY_DATA` guards in MSIR/PCGIR/PoseSearch/test files, and the ticket-named `MCP_TEST_HAS_CREATE_DELEGATE*` gates in Blueprint/BPIR tests. Test: `rg -n '#\s*if\s+WITH_EDITOR\b|#\s*ifn?def\s+WITH_EDITOR\b|!WITH_EDITOR\b'`, `rg -n 'WITH_EDITORONLY_DATA'`, and `rg -n 'MCP_TEST_HAS_CREATE_DELEGATE|MCP_TEST_HAS_CREATE_DELEGATE_GUID'` under `C:/Unity/unreal-fpv-pluginwork/Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway`.
- `#5-remaining-guards-removed` `IN-REVIEW` developer — Removed the remaining editor-only guards from the assigned AI, PoseSearch, MSIR, PCGIR, MetaSound, and test files; stripped the local CreateDelegate test gates and hardcoded AI feature defines while preserving optional header probes, optional MetaSound search engine gates, and dev automation gates. Verified after this fix with static `rg` sweeps; no build or tests run.
- `#6-verify-guards-gone` `DONE` tester — Verified: re-ran the three `rg` sweeps from `#4` (`#\s*if\s+WITH_EDITOR\b|#\s*ifn?def\s+WITH_EDITOR\b|!WITH_EDITOR\b`, `WITH_EDITORONLY_DATA`, and `MCP_TEST_HAS_CREATE_DELEGATE|MCP_TEST_HAS_CREATE_DELEGATE_GUID`) under `Source/EditorAutomationRpcGateway`; all three returned zero matches, and `AIHandler.cpp` is also clean.
- `#7-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 10 body citations repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
