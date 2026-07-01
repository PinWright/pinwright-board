---
id: B-compile-save-after-compile-timeout
title: "blueprint.compile saveAfterCompile times out the MCP call on Widget BPs though the save succeeds"
status: WONTFIX
severity: Medium
category: bug
tags: [blueprint, compile, save, widget, timeout, false-negative, jobs]
---

# blueprint.compile saveAfterCompile times out the MCP call on Widget BPs though the save succeeds

`blueprint.compile {saveAfterCompile:true}` on a Widget Blueprint ran past the
MCP client timeout and surfaced "The operation timed out", but the asset was
actually written to disk (confirmed by `git status` showing the `.uasset`
modified afterward). The save is synchronous on the game thread with no job
ticket: `BlueprintCompileHandler.cpp:53` calls `SaveLoadedAssetThrottled(BP)`,
which (`AssetUtils.cpp:498`) calls `UEditorAssetLibrary::SaveLoadedAsset(Asset)`
inline; the handler only reaches `Ctx.SendSuccess` (`BlueprintCompileHandler.cpp:80`)
after the save returns. A slow Widget-BP save (thumbnail regeneration plus the
CEF-backed UMG host) can exceed the request budget (`HttpDefaultTimeoutMs=120000`,
`HttpMaxTimeoutMs=300000` in `EditorAutomationRpcGatewaySettings.cpp:23-24`),
so the caller gets a false-negative error and cannot tell whether to retry.

Note: `blueprint.compile` was NOT among the 20 handlers migrated to the job
system in `F-long-running-tickets` (its "Affected handlers" list omits it), so
unlike `editor.save_all` it has no ticket / `system.job_status` path. The
sync-budget option discussed in `E-save-all-sync-fast-path` (#1) is the natural
fix vehicle here.

**Workaround:** Compile without `saveAfterCompile`, then save out-of-band (or
re-query / check `git status`) to confirm the write; don't trust the timeout as
a real failure.
**Fix:** Route the `saveAfterCompile` branch through the job-ticket system
(`Ctx.StartJob` → poll `system.job_status`), or bound the inline save with a
sync-budget fast-path and report a partial-success / pending status instead of
misreporting a completed save as a timeout.

## History
- `#1-widget-bp-save-timeout-false-negative` `OPEN` reporter — `blueprint.compile {path:"/App/App/UI/LobbyAndMenu/HUD/W_HUD_PhotoInspection", saveAfterCompile:true}` returned MCP "The operation timed out"; a prior no-save `blueprint.compile` on the same asset returned `{compiled:true, status:"UpToDate", saved:false}` well under the timeout (compile is fast — the save blocks), and `git status --short` afterward showed `M Plugins/App/Content/App/UI/LobbyAndMenu/HUD/W_HUD_PhotoInspection.uasset` (save completed despite the error). Sync save at `BlueprintCompileHandler.cpp:53` → `SaveLoadedAssetThrottled` → `UEditorAssetLibrary::SaveLoadedAsset` (`AssetUtils.cpp:498`), no job ticket. Distinct from `F-long-running-tickets` (blueprint.compile not migrated) and `E-save-all-sync-fast-path` (editor.save_all, opposite direction).
- `#2-wontfix-maintainer` `WONTFIX` maintainer — Closed WONTFIX by maintainer. The save itself succeeds (persists to disk); only the timeout message is misleading, and the workaround (compile without `saveAfterCompile`, save/verify out-of-band) is cheap, so the async/sync-budget rework isn't worth it.
