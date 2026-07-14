---
id: F-editor-set-fixed-delta-time-proper
title: "Reimplement editor.set_fixed_delta_time via FApp::SetFixedDeltaTime + SetUseFixedTimeStep (removed as a bogus-cvar stub)"
status: OPEN
severity: Medium
category: feature
tags: [editor, fixed-timestep, determinism, reimplement, rpc-cull]
---

# Reimplement `editor.set_fixed_delta_time` properly (fixed timestep)

`editor.set_fixed_delta_time` was **removed** in the RPC cull recorded in
[`E-rpc-cull-151-record`](E-rpc-cull-151-record.md) because it was a stub that
executed a console variable that does not exist. Pinning a fixed simulation
delta time is a real, wanted capability (deterministic capture, repeatable
physics stepping, frame-locked recording) with no RPC alternative.

## What the removed version did wrong

The old handler (`Handlers/Editor/EditorCommandHandler.cpp:662-667`) ran
`GEditor->Exec(..., "r.FixedDeltaTime <value>")`, ignored the return, and
unconditionally returned `success:true` echoing the requested `deltaTime`. There
is **no `r.FixedDeltaTime` cvar anywhere in UE** (verified across Runtime,
Editor, and Plugins) — the engine's fixed-timestep mechanism is
`FApp::SetFixedDeltaTime` / `FApp::SetUseFixedTimeStep`, never a cvar. So the
exec was a guaranteed no-op reported as success.

## Proper implementation

- On enable: `FApp::SetFixedDeltaTime(DeltaTime)` then
  `FApp::SetUseFixedTimeStep(true)`.
- Provide a way to disable (e.g. a `bEnabled`/`useFixedTimeStep` param, or a
  companion clear) that calls `FApp::SetUseFixedTimeStep(false)` and restores
  real-time stepping — leaving the editor stuck in fixed-timestep is a footgun.
- Validate `deltaTime > 0`.
- Read back and return the actual `FApp::GetFixedDeltaTime()` /
  `FApp::UseFixedTimeStep()` so the response reflects real engine state, not an
  echo.

**Fix:** New handler in `EditorCommandHandler.cpp` (16 other handlers remain).
Regression test asserts `FApp::GetFixedDeltaTime()` equals the requested value
and `FApp::UseFixedTimeStep()` is true after the call, and that the disable path
clears it (restore state in test teardown so it does not leak into sibling
tests).

## History
- `#1-reimpl-after-cull` `OPEN` reporter — Filed to reinstate the wanted capability removed by the RPC cull ([`E-rpc-cull-151-record`](E-rpc-cull-151-record.md)). The removed `editor.set_fixed_delta_time` execed a nonexistent cvar `r.FixedDeltaTime` (EditorCommandHandler.cpp:662-667) and reported success regardless. Proper impl: `FApp::SetFixedDeltaTime` + `FApp::SetUseFixedTimeStep`, with a disable path and real-state readback. No RPC alternative exists for fixed-timestep control.
