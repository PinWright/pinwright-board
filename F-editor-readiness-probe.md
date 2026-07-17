---
id: F-editor-readiness-probe
title: "No readiness probe: python.execute hard-fails PYTHON_INIT_FAILED during the editor-startup window"
status: OPEN
severity: Medium
category: feature
tags: [system, python, startup, readiness, ergonomic]
---

# No readiness probe: python.execute hard-fails PYTHON_INIT_FAILED during the editor-startup window

The PinWright HTTP port begins accepting connections (and the transport-level
`ping` answers `{}`) well before the editor's subsystems are usable. There is a
~30 s window right after launch where the port is up but `python.execute` hard-
fails, and **no RPC exists to tell an agent when subsystems are actually ready**,
so agents are forced to sleep blind before their first real call.

## The startup race (python.execute)

On UE ≥ 5.6, `python.execute` checks readiness once and fails hard with no retry
or bounded wait. In `Handlers/System/PythonExecuteHandler.cpp` (lines 127-137):

```cpp
if (!Python->IsPythonInitialized())
{
    Python->ForceEnablePythonAtRuntime();      // line 129 — single attempt

    if (!Python->IsPythonInitialized())
    {
        Ctx.SendError(TEXT("PYTHON_INIT_FAILED"),  // line 133 — hard fail, no retry
            TEXT("Python could not be initialized. Check Output Log for details."));
        return true;
    }
}
```

There is exactly one `ForceEnablePythonAtRuntime()` call and one re-check; if the
engine's own async python startup hasn't finished, the handler returns
`PYTHON_INIT_FAILED` immediately. Session evidence (2026-07-17): two `python.execute`
calls right after launch failed with `[PYTHON_INIT_FAILED] Python could not be
initialized. Check Output Log for details.`; the log shows the handler calling
`ForceEnablePythonAtRuntime` from `PythonExecuteHandler.cpp(129)` at 08:00:48 and
failing, while engine python startup scripts only finished at ~08:01:17 — a ~30 s
window where the port is up but `python.execute` cannot succeed.

## No readiness signal to poll

The agent probed for a readiness RPC and found none:
`call {method: "system.health"}` → `[UNKNOWN_ACTION] Unknown action: system.health.
Did you mean: system.job_list, system.run_ubt, ...`. Nothing else fills the gap:

- The `system.*` namespace exposes `system.job_status` / `job_list` / `job_cancel`,
  `system.run_ubt` / `run_tests` / `console_command`, `system.console.search`,
  `system.live_coding_status` / `live_coding_compile`, and the `system.inspect.*`
  read-only inspectors — none report subsystem-init state.
- `editor.status` (see `F-editor-status`, DONE) is a read-only probe but reports
  **PIE / editor-world** state, not whether python or other subsystems finished
  initializing.
- The transport-level `ping` returns `{}` unconditionally (handled in the transport,
  not the dispatcher), so it confirms the socket is alive but says nothing about
  subsystem readiness.

Because no signal distinguishes "starting" from "ready", the agent's workaround was
a hardcoded **40 s sleep** after port-up on the next launch — blind, wasteful on a
fast machine, and still racy on a slow one.

**Relation to `B-proxy-blocks-minutes-on-hung-editor`:** that ticket's proxy-side
`ping` liveness probe already lumps "starting" in with "hung/busy" — a real
subsystem-readiness signal would let the proxy and agents tell a still-initializing
editor apart from a genuinely hung one instead of guessing from a timeout.

**Workaround:** blind fixed sleep (~40 s) after the port opens, then retry
`python.execute`. Racy on slow machines, wasteful on fast ones.

**Fix (maintainer-directed architecture):** C++ exposes only a cheap read-only
readiness state (e.g. `system.health` reporting subsystem-init flags incl. python via
`IPythonScriptPlugin::Get()->IsPythonInitialized()` on UE ≥ 5.6 / `IsPythonAvailable()`
on 5.4-5.5). The **blocking/wait lives in the python proxy**
(`Content/Python/mcp_proxy.py`), not in C++ handlers: the proxy holds the caller's
request (SSE/long-poll style) until the editor reports ready, because only the proxy
can also span the earlier boot phase where the HTTP server isn't bound yet
(connection refused) — a window no in-editor RPC can cover. Do NOT add blocking
waits inside C++ handlers.

## History
- `#1-startup-init-race` `OPEN` reporter — Filed from a live session (2026-07-17): the port accepts connections ~30 s before python is usable; two `python.execute` calls failed `PYTHON_INIT_FAILED` while engine python startup was still running (handler at `PythonExecuteHandler.cpp(129)` at 08:00:48, engine python ready ~08:01:17). Verified in source: `PythonExecuteHandler.cpp` lines 127-137 make a single `ForceEnablePythonAtRuntime()` attempt with no retry/bounded wait, hard-failing at line 133. No readiness RPC exists — `system.health` → `UNKNOWN_ACTION`; enumerated `system.*` methods, `editor.status` (PIE-state only), and transport `ping` (`{}` regardless) none report subsystem-init state. Agent workaround was a blind 40 s sleep. Severity Medium — soft blocker, only via a blind-sleep workaround; reach is every cold-start session that uses python early.
- `#2-proxy-owns-the-wait` `OPEN` maintainer — Ruling: readiness RPC should be added, but the blocking belongs in the python proxy ("sse blocking into py, not into c++"). Fix section rewritten: C++ = cheap readiness state only; `mcp_proxy.py` holds/blocks the request until ready and is the only layer that can also cover the pre-bind connection-refused boot phase. Clarified against the report's framing: port 24281 is served by the plugin's C++ HTTP server, python is a separately-initializing engine subsystem behind one handler — the 30 s port-up-python-down window is real (C++-origin `PYTHON_INIT_FAILED` served over the port while engine python scripts were still running). See also `F-proxy-editor-start-rpc` (proxy-side editor launch, same ruling).
