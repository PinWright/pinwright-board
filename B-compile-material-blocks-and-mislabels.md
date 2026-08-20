---
id: B-compile-material-blocks-and-mislabels
title: "compile_material blocks the game thread for a whole cold shader compile with no job handle, and the stall is reported to every other caller as a retryable EDITOR_NOT_READY — the accurate EDITOR_GAME_THREAD_STALLED result is produced and then discarded by the proxy"
status: OPEN
severity: High
category: bug
tags: [material, material-authoring, compile_material, game-thread-block, shader-compile, watchdog, EDITOR_NOT_READY, EDITOR_GAME_THREAD_STALLED, job-handle, concurrency, wrong-error-code]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# A four-minute synchronous block that tells everyone else the editor is still starting up

## The block

`Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3476` calls
`MaterialCompileErrorCollector::WaitAndCollect`, which is
`Source/PinWright/Private/Handlers/Material/MaterialCompileErrorCollector.h:46-51`:

```cpp
Material->CacheShaders(EMaterialShaderPrecompileMode::Synchronous);   // :46
...
GShaderCompilingManager->FinishAllCompilation();                      // :51
```

Both run on the game thread, unbounded, with no progress, no timeout and no cancel. On a fresh
master's first compile that is minutes — ~4 measured. Before the wait, `:3468`
(`ApplyMasterMaterialEdit`) has already built an `FMaterialUpdateContext` with
`RecreateRenderStates | SyncWithRenderingThread`, which flushes rendering commands in both its
constructor and destructor, plus `UpdateAllComponentMaterialInstances(true)` per consuming landscape.

There is **no job handle**: `grep StartJob` over `Source/PinWright/Private/Handlers/Material/`
returns zero hits; the handler is straight-line to `Ctx.SendSuccess` at `:3531`. And `wait: false`
does not help — `bWaitAccepted` (`Transport/McpRequestCore.cpp:810-816`, consumed at
`Transport/SocketHttpServer.cpp:997-1002`) only selects ticket-vs-SSE for the HTTP response. It frees
the HTTP caller; the handler stays synchronous on the game thread and the block is unchanged.

## The error code is wrong, and the right one is thrown away

Past `GameThreadStallReportSeconds` (default **90.0 s**, `PinWrightSettings.cpp:43`) the I/O-thread
probe sets `bGameThreadStalled` (`Transport/SocketHttpServer.cpp:867-874`). That folds into
`editorReady: false` (`Transport/McpRequestCore.cpp:380-381`), and `:411` emits the dedicated
`EDITOR_GAME_THREAD_STALLED` **with `inFlightMethod`** — precisely the diagnosis a caller needs.

The proxy then discards it. `Content/Python/mcp_proxy.py:1596-1601` (`_probe_state`) checks
`blockedOnModal`, then simply tests `"editorReady" in result` and returns `"not_ready"`. It never
inspects `gameThreadStalled`. `"not_ready"` maps to `EDITOR_NOT_READY` (`:3067`, `:3089`) and is
marked `retryable: True` (`:1699`).

So from t≈90 s every other client is told the editor is **still starting up and worth retrying**,
while it is in fact executing their own previous call — which invites exactly the poll loop that
keeps the watchdog hot. A wrong condition, named confidently, on a normal path.

## Concurrency is refused, not queued

`mcp_proxy.py:3057-3076` probes before forwarding and returns `_editor_unavailable_result(...)` for
any non-`alive` state, so a second compile is rejected outright. The in-editor gate
(`McpRequestCore.cpp:612`) would also refuse, though note it tests
`!IsEditorOperational(Config) || bBlockedOnModal` and does **not** include `bGameThreadStalled`.

## What is and is not already tracked

`B-compile-material-not-tick-gated` (OPEN, High, encounters 3) covers the **tick-safety** half — the
render flush and live render-state recreation from the handler body, with the verb absent from
`GTickUnsafeMethodNames`. Confirmed still absent: `grep compile_material Dispatch/SafePoint.cpp`
returns one comment hit at `:143` and no table entry. That ticket does **not** cover the unbounded
block, the missing job handle, the mislabelled retryable error, or the concurrency refusal — this
ticket is those four.

`B-compile-material-false-shader-success` (DONE) is the ticket whose fix *introduced*
`WaitAndCollect` / `FinishAllCompilation`. The honesty it bought is worth keeping; the synchronous
shape it bought is what this ticket asks to change.

**Fix:** give the verb a real job handle so a long compile returns a ticket and progresses through
`system.job_status` instead of holding the game thread — the transport already carries the
ticket/SSE machinery, only the handler is synchronous. Independently and much cheaper: teach
`_probe_state` to read `gameThreadStalled` and surface `EDITOR_GAME_THREAD_STALLED` with its
`inFlightMethod` instead of collapsing to a retryable `EDITOR_NOT_READY`; that one change turns an
unexplained four-minute outage into a message naming the call responsible.

## Related

- `B-compile-material-not-tick-gated` (OPEN, High) — the tick-safety half.
- `B-compile-material-false-shader-success` (DONE) — origin of the synchronous wait.
- `B-compile-material-landscape-consumers-stale`, `B-landscape-set-material-stale-mics`,
  `B-material-authoring-save-no-disk-write` — same verb, other axes.

## History
- `#1-unbounded-block-and-a-retryable-lie` `OPEN` reporter — `material.authoring.compile_material` blocks the game thread for the full duration of a cold shader compile: `MaterialAuthoringHandler.cpp:3476` calls `MaterialCompileErrorCollector::WaitAndCollect`, which runs `CacheShaders(EMaterialShaderPrecompileMode::Synchronous)` (`MaterialCompileErrorCollector.h:46`) then `GShaderCompilingManager->FinishAllCompilation()` (`:51`), unbounded and uncancellable, after `:3468` has already flushed rendering commands twice per update context and rebuilt every consuming landscape's MICs. No job handle exists (`grep StartJob` over `Handlers/Material/` = 0 hits; straight-line to `SendSuccess` at `:3531`), and `wait:false` does not help because `bWaitAccepted` (`McpRequestCore.cpp:810-816` → `SocketHttpServer.cpp:997-1002`) only selects ticket-vs-SSE for the HTTP response and leaves the handler synchronous. Past `GameThreadStallReportSeconds` (default 90 s, `PinWrightSettings.cpp:43`) the probe sets `bGameThreadStalled` (`SocketHttpServer.cpp:867-874`) → `editorReady:false` (`McpRequestCore.cpp:380-381`), and `:411` emits the dedicated `EDITOR_GAME_THREAD_STALLED` carrying `inFlightMethod` — which the proxy then throws away: `_probe_state` (`mcp_proxy.py:1596-1601`) tests only for the presence of `editorReady` and never reads `gameThreadStalled`, collapsing to `"not_ready"` → `EDITOR_NOT_READY` (`:3067`, `:3089`) marked `retryable: True` (`:1699`). Every other client is therefore told the editor is still starting up and worth retrying while it is executing their own previous call, which invites the poll loop that keeps the watchdog hot. Concurrency is refused rather than queued (`mcp_proxy.py:3057-3076` probes before forwarding; the in-editor gate at `McpRequestCore.cpp:612` also refuses, and notably does not include `bGameThreadStalled` in its condition). `B-compile-material-not-tick-gated` (OPEN, High) covers only the tick-safety half — the verb is still absent from `GTickUnsafeMethodNames` (`Dispatch/SafePoint.cpp:143` is a comment, not an entry) — and does not cover the block, the missing job handle, the mislabelled error or the refusal. Fix: give the verb a job handle so the transport's existing ticket/SSE path can carry it, and separately teach `_probe_state` to surface `EDITOR_GAME_THREAD_STALLED` with `inFlightMethod` — the cheap half that turns an unexplained four-minute outage into a message naming the responsible call.
