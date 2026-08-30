---
id: B-asset-dump-folder-no-completion-signal
title: "asset.dump_folder completion not reported through job registry"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, asset, dump-folder, no-completion-signal]
---

# asset.dump_folder completion not reported through job registry

`asset.dump_folder` already used an async ticker-based approach (see `F-asset-dump-text-mirror` / `#4-async-folder-dump-rework`) and returned a status companion RPC. However its completion signal was not integrated into the job registry, so it did not appear in `system.job_list` and did not write a JSONL event to `jobs.jsonl`.

**Fix:** The existing `FAsyncFolderDumpState` now calls `Ctx.StartJob()` at kickoff time instead of rolling a custom status mechanism. The per-tick body emits progress through `FJobRegistry::RecordProgress` (rate-capped at 1 Hz). `FinalizeFolderDump` calls `FJobRegistry::Complete` when the sweep ends. The legacy `asset.dump_folder_status` RPC was removed in favour of the unified `system.job_status`.

**Files:** `Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp`, `Private/State/AsyncFolderDumpState.h`, `docs/asset-dump.md`.

## History
- `#1-not-in-registry` `OPEN` reporter — `asset.dump_folder` had its own completion state but was invisible to `system.job_list` and wrote nothing to `jobs.jsonl`.
- `#2-integrated-into-job-registry` `IN-REVIEW` developer — `FAsyncFolderDumpState` migrated to use `Ctx.StartJob()`. `FinalizeFolderDump` calls `FJobRegistry::Complete`. Progress events emitted at 1 Hz from per-tick body.
- `#3-status-shim-removed` `IN-REVIEW` developer — `asset.dump_folder_status` RPC deleted; callers use `system.job_status` with the ticket id returned from kickoff. `docs/asset-dump.md` updated to document the ticket flow.
- `#4-verified-end-to-end` `DONE` tester — Verified: kicked `asset.dump_folder /App/HELIOS/Drones/Icarus recursive:true` → ticket `j_20260427T024556_9731c015`, response had `{ticket_id, monitor_path, method, started_at, rootDir, folderPath, assetCount:33}`. `system.job_list` showed it active. `jobs.jsonl` recorded `event:"started"`, multiple `event:"progress"` lines (`"dumped N / 929"` rate-capped at 1Hz), and `event:"completed"` with `{result:{rootDir, dumped:929}}`. `system.job_status` after completion returned `status:"completed"`. End-to-end registry integration confirmed.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
