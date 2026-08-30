---
id: F-test-progress-protocol-removed
title: "Remove dead system.test_progress_protocol and system.test_stale_progress handlers"
status: DONE
severity: Low
category: feature
tags: [cleanup, dead-code, system, test-scaffolding]
---

# Remove dead system.test_progress_protocol and system.test_stale_progress handlers

`system.test_progress_protocol` and `system.test_stale_progress` were temporary scaffolding handlers added during early development to validate the progress-reporting approach. They were never documented in the tool catalog as production RPCs, but remained registered and visible in `"system?"` discovery output, adding noise.

Both handlers were removed as part of the job-ticket migration. Callers that previously used `system.test_progress_protocol` to validate the async signalling path should instead use `system.job_status` against a real job, or inspect `jobs.jsonl` directly. No callers of these methods appear in the PDS project codebase.

**Files:** Deleted `Source/EditorAutomationRpcGateway/Private/Handlers/System/TestProgressProtocolHandler.cpp` and `TestStaleProgressHandler.cpp`.

## History
- `#1-dead-scaffolding-handlers` `OPEN` reporter — `system.test_progress_protocol` and `system.test_stale_progress` are undocumented test-scaffolding handlers that appear in `"system?"` discovery output as noise; they have no production callers and should be removed.
- `#2-handlers-removed` `IN-REVIEW` developer — Both handler `.cpp` files deleted. `REGISTER_RPC_HANDLER` macros were the only registration path, so deletion is sufficient — no other changes required. Discovery output for `"system?"` confirmed clean post-deletion.
- `#3-verified-unknown-action` `DONE` tester — Verified: raw RPC `{method:"system.test_progress_protocol"}` returned `{error:{code:"UNKNOWN_ACTION", message:"Unknown action: system.test_progress_protocol"}}`; same for `system.test_stale_progress`. Discovery `"system?"` returned 19 production methods, none matching either removed handler.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. **Deliberately not repointed, and correct as filed.** The `**Files:**` line reads “Deleted `…TestProgressProtocolHandler.cpp` and `TestStaleProgressHandler.cpp`”, so naming absent files is the record, not an error. Both are absent at HEAD **and** absent from the plugin's visible git history — the deletion predates root commit `17a331d7`. Rewriting the module root would produce a `Source/PinWright/…` path that is equally nonexistent and more misleading. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
