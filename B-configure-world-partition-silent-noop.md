---
id: B-configure-world-partition-silent-noop
title: "performance.configure_world_partition is a silent no-op that echoes inputs as effective settings (targets CVars that don't exist in UE 5.7)"
status: IN-REVIEW
severity: High
category: bug
tags: [performance, world-partition, cvar, silent-failure, no-op]
---

# performance.configure_world_partition applies nothing and reports success with the requested values

`performance.configure_world_partition` (`PerformanceHandler.cpp` lines
684–729) claims to "Configure World Partition streaming settings", but in
UE 5.7 it applies **nothing** and returns a success payload that simply
echoes the caller's inputs back as if they were the effective grid settings.

The handler tries to set three console variables:

```cpp
SetCVar(TEXT("wp.Runtime.EnableStreaming"), bEnabled ? 1 : 0);
if (bHasCellSize)
    SetCVarFloat(TEXT("wp.Runtime.RuntimeCellSize"), (float)CellSize);
if (bHasLoadingRange)
    SetCVarFloat(TEXT("wp.Runtime.RuntimeStreamingRange"), (float)LoadingRange);
```

All three of these CVar names are wrong / nonexistent in UE 5.7:

- `wp.Runtime.EnableStreaming` — does not exist (`system.console.search query="EnableStreaming"` returns only `pf.EnableStreaming`, `GroomCache.EnableStreaming`, `wp.Editor.WorldExtentToEnableStreaming`).
- `wp.Runtime.RuntimeCellSize` — does not exist (`system.console.search query="wp.Runtime.RuntimeCellSize"` → `{"results":[],"totalMatches":0}`). RuntimeCellSize is a per-grid editor property on `UWorldPartitionRuntimeSpatialHash`, not a runtime CVar.
- `wp.Runtime.RuntimeStreamingRange` — does not exist. The closest real CVar is `wp.Runtime.OverrideRuntimeLoadingRange`.

The `Set` helpers guard with `FindConsoleVariable`, which returns `nullptr`
for all three, so every `Set()` is silently skipped — nothing is applied.
But the response is built from the **request**, not from any readback:

```cpp
Resp->SetBoolField(TEXT("streamingEnabled"), bEnabled);   // echoes the 'enabled' input (default true)
if (bHasCellSize)    Resp->SetNumberField(TEXT("cellSize"), CellSize);     // echoes the input
if (bHasLoadingRange) Resp->SetNumberField(TEXT("loadingRange"), LoadingRange); // echoes the input
```

So a caller gets `{streamingEnabled:true, cellSize:25600, loadingRange:51200}`
and reasonably concludes the grid is now configured that way. It is not —
none of the three values were applied to anything.

**Why it matters:**

- This is exactly the task an agent is asked to do: "turn on WP runtime
  streaming, set the runtime cell size and loading range, then tell me the
  settings now in effect so I can confirm." The response *falsely confirms*
  the configuration. There is no JSON-level signal that the call did nothing.
- The echo is provably an echo, not a readback: calling with `{}` returns
  only `{"streamingEnabled":true}` (no cellSize/loadingRange), and calling
  with arbitrary garbage `{cellSize:99999, loadingRange:88888}` returns
  `{streamingEnabled:true, cellSize:99999, loadingRange:88888}` — the result
  always mirrors whatever was passed in.
- WP runtime streaming enable is a world/level property
  (`UWorldPartition::bEnableStreaming`) — there is a separate, real handler
  `level.structure.enable_world_partition` for the toggle — and cell size is
  a per-grid `UWorldPartitionRuntimeSpatialHash` property, neither of which is
  reachable through these (nonexistent) CVars.

**Fix options (any of):**

1. **Apply the real settings + read back.** Resolve the active world's
   `UWorldPartition` / `UWorldPartitionRuntimeSpatialHash`, set the actual
   grid `CellSize` / loading-range properties (and `bEnableStreaming`), call
   the appropriate post-edit, then build the response from the **observed**
   values, not the request. For loading range, the real runtime override CVar
   is `wp.Runtime.OverrideRuntimeLoadingRange`; verify each
   `FindConsoleVariable` succeeded and report what was actually set.
2. **Fail loud.** If a target CVar/property is missing (e.g. no World
   Partition on the active world, or `FindConsoleVariable` returns nullptr),
   `SendError("NOT_APPLIED"/"CVAR_NOT_FOUND", …)` instead of `SendSuccess`,
   so callers can distinguish "applied" from "ignored". At minimum stop
   echoing inputs that were never written.

Option 1 closes the capability; option 2 is the minimum honesty fix.

## Repro

1. `performance.configure_world_partition {enabled:true, cellSize:25600, loadingRange:51200}`
   → `{streamingEnabled:true, cellSize:25600, loadingRange:51200}` (no error).
2. `system.console.search {query:"wp.Runtime.RuntimeCellSize"}` → `{"results":[],"totalMatches":0}`.
   `system.console.search {query:"EnableStreaming"}` → no `wp.Runtime.EnableStreaming`.
   The three CVars the handler sets do not exist → every `CVar->Set()` is a silent no-op.
3. `performance.configure_world_partition {}` → `{"streamingEnabled":true}` (proves the
   reported `streamingEnabled` is the echoed `enabled` default, not a readback).
4. `performance.configure_world_partition {cellSize:99999, loadingRange:88888}` →
   `{streamingEnabled:true, cellSize:99999, loadingRange:88888}` (proves cellSize/loadingRange
   are echoes of the request, not effective state).

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `performance.configure_world_partition {enabled:true, cellSize:25600, loadingRange:51200}` → `{streamingEnabled:true, cellSize:25600, loadingRange:51200}` with no error. Confirmed via `system.console.search` that all three target CVars (`wp.Runtime.EnableStreaming`, `wp.Runtime.RuntimeCellSize`, `wp.Runtime.RuntimeStreamingRange`) do not exist in UE 5.7, so `FindConsoleVariable` returns nullptr and every `Set()` in `PerformanceHandler.cpp:712-718` is silently skipped. Response is built from the request, not a readback: `{}` → `{streamingEnabled:true}`, `{cellSize:99999,loadingRange:88888}` → echoes 99999/88888. Net: silent success-with-no-effect that falsely confirms the requested grid configuration. No existing board ticket for this method (ripgrep + qmd dedup clean).
- `#2-fail-loud-fix` `IN-REVIEW` developer — Implemented Fix Option 2 (fail loud, the established house pattern matching DONE `B-material-stub-handlers-silent-success`). Changed the `SetCVar`/`SetCVarFloat` lambdas in `PerformanceHandler.cpp` to return whether the target CVar was actually resolved+set, collect any unresolved names into `MissingCVars`, and `SendError("CVAR_NOT_FOUND", …, {missingCVars:[…]})` instead of building a fabricated success payload when one or more targets are missing. The error message points callers at the real handlers (`level.structure.configure_grid_size` for cell size / loading range, `level.structure.enable_world_partition` for the streaming toggle). On UE 5.7 all three CVars are absent so the call now fails loud instead of fake-confirming. Added explicit `Dom/JsonObject.h`/`Dom/JsonValue.h` includes for the error-data array. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Debug/PerformanceHandler.cpp`. Regression test: replaced the bare no-crash assertion with `FPerfConfigureWorldPartitionFailsLoudTest` (`EditorAutomationRpcGateway.performance.configure_world_partition.MissingCVarsFailLoud`) in `Source/EditorAutomationRpcGateway/Private/Tests/EditorOps/TestDebugHandlers.cpp` — it invokes the production handler with capture and asserts `bSuccess==false` + `ErrorCode=="CVAR_NOT_FOUND"` and that no echoed `cellSize` surfaces; reverting to the silent `SendSuccess` echo makes it fail. (Not compiled/tested here — later phase drives green.)
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
