---
id: E-blueprint-compile-compile-only
title: "Make blueprint.compile compile-only; remove saveAfterCompile (superseded by asset.save)"
status: DONE
severity: Low
category: ergonomic
tags: [blueprint, compile, save, breaking-change]
---

# Make `blueprint.compile` compile-only; remove `saveAfterCompile`

`blueprint.compile` carries a `saveAfterCompile` boolean
(`BlueprintCompileHandler.cpp:17,49-59`) that, on compile success, runs the
integrity gate and saves via `SaveLoadedAssetThrottled`. This conflates two
concerns — compile-status check vs. persistence — on one verb, and the save
half becomes redundant once a generic `asset.save` exists (`F-asset-save`).

It also does not carry the corruption protection it appears to: the integrity
gate lives in the save helper itself (`SaveLoadedAssetThrottled`,
`AssetUtils.cpp:588-602`), so it fires on
ANY save path (`asset.save`, `editor.save_all`), not just this parameter.
Removing `saveAfterCompile` therefore loses no protection — only the inline
integrity verdict in the compile response, which moves to `asset.save`. In that
sense the flag does nothing `asset.save` won't do better.

Remove the `saveAfterCompile` param and the `bSuccess && bSaveAfterCompile`
branch; `blueprint.compile` becomes a pure compile-status call. Keep the
integrity gate on the save path (validate-on-save — the universal choke point;
compile shouldn't pay for a graph-integrity walk when nothing is persisted).

Ordering: land `F-asset-save` first so the targeted single-BP save path isn't
lost. Once removed, the WONTFIX `B-compile-save-after-compile-timeout` is moot
for this verb (the sync-save-timeout concern, if it recurs, belongs to
`asset.save`).

**Workaround:** n/a — deliberate API change.
**Fix:** Delete `saveAfterCompile` from the `blueprint.compile` ParamSpec and the
save branch (`BlueprintCompileHandler.cpp`), plus the now-unused inline
integrity-reporting there (it moves to `asset.save`). Breaking: callers passing
`saveAfterCompile:true` must call `asset.save` after compiling.

## History
- `#1-decouple-compile-from-save` `OPEN` reporter — `saveAfterCompile` (`BlueprintCompileHandler.cpp:17,49-59`) couples persistence to compile and is redundant once `F-asset-save` lands; the integrity gate it appears to "carry" actually lives in `SaveLoadedAssetThrottled` (`AssetUtils.cpp:588-602`) and protects all save paths regardless. Remove the param, keep compile pure, retain validate-on-save. Depends on `F-asset-save`; supersedes the WONTFIX `B-compile-save-after-compile-timeout` for this verb.
- `#2-reword-stale-integrity-cite` `OPEN` developer — Corrected the integrity-gate citation in the body and History #1 from the stale `AssetUtils.cpp:481-495` (those lines are inside `McpSafeLevelSave`, level-save backoff) to the real gate at `AssetUtils.cpp:588-602` (the `Cast<UBlueprint>` → `ValidateBlueprintGraphIntegrity` refuse-to-write block inside `SaveLoadedAssetThrottled`) — matching the corrected cite the sibling `F-asset-save` already carries. Substance unchanged; the param/branch (`BlueprintCompileHandler.cpp:17,49-59`) and the `asset.save` replacement (`AssetSaveHandler.cpp`) verified present in source under test.
- `#3-implement-compile-only` `IN-REVIEW` developer — Made `blueprint.compile` compile-only. Removed the `saveAfterCompile` ParamSpec (`RPC_PARAM_DEF`) and the `bSuccess && bSaveAfterCompile` save branch, the payload parse, and the now-unused inline integrity-gate reporting (`integrityFailures`/`integrityGate:"blocked"`/`integrityMessage`) from `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BlueprintCompileHandler.cpp`; the response now omits the `saved` field. Validate-on-save is retained at the universal choke point (`SaveLoadedAssetThrottled`, `AssetUtils.cpp:588-602`) and surfaced inline by `asset.save` (`AssetSaveHandler.cpp`). Dropped the now-stale `saveAfterCompile:false` line from the existing passive test (`Tests/Blueprint/TestBlueprintHandlers.cpp`). Added regression test `EditorAutomationRpcGateway.blueprint.compile.NoSaveAfterCompileParam` (same file) asserting the live `blueprint.compile` ParamSpec carries `path` but NOT `saveAfterCompile` — fails if the param were re-added. Breaking: callers passing `saveAfterCompile:true` must call `asset.save` after compiling. Not compiled/tested here (later phase).
- `#4-verified-compile-only-live` `DONE` tester — Verified against source + live editor. `BlueprintCompileHandler.cpp:14-46` is now pure compile-only: ParamSpec carries only `path` (+ aliases `blueprintPath`/`assetPath`/...), no `saveAfterCompile`; the handler body has no `bSaveAfterCompile` save branch and no inline integrity reporting (the comment at lines 34-37 states persistence + its gate moved to the save path / `SaveLoadedAssetThrottled`). Regression test present and asserts the contract: `FBlueprintCompileNoSaveParamTest` (`Tests/Blueprint/TestBlueprintHandlers.cpp:350-363`, `PinWright.blueprint.compile.NoSaveAfterCompileParam`) — `path` required, `saveAfterCompile` absent. Live session evidence: `blueprint.compile {saveAfterCompile:true}` on `/App/App/UI/LobbyAndMenu/W_LyraFrontEnd` → `[UNKNOWN_PARAMS] Unknown parameter(s) ... [saveAfterCompile]` (param correctly gone from the running binary); `asset.save` fallback → `{saved:true, sizeBytes:1060243}`, no `integrityGate:"blocked"` (save-choke-point gate ran clean). Live wiki `wiki-src/blueprint.md:384` already announces the removal and redirects callers to `asset.save`. Also disproved a proposed bug `B-blueprint-compile-saveaftercompile-param-gone` ("dead/unreachable save-gate code path"): source shows the save+gate code was **fully removed**, not orphaned, so no dead path exists and the gate stays reachable via `asset.save` — not filed, subsumed here. (The proposed entry's other premises were inaccurate: `blueprint.bpir-gotchas.md` does NOT reference `saveAfterCompile`, and the remaining `saveAfterCompile:true` mentions live only in append-only, immutable History entries such as `B-bp-saved-state-corruption-mcp-edits` #4/#8.)
