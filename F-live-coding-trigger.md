---
id: F-live-coding-trigger
title: "system: in-process Live Coding compile trigger + status readback"
status: IN-REVIEW
severity: Medium
category: feature
tags: [system, live-coding, build, parity-ue58]
---

# system: in-process Live Coding compile trigger + status readback

PinWright has no Live Coding RPC (grep `LiveCoding` across `Source\` = zero). `system.run_ubt` spawns UBT as an external child process, which is the wrong tool mid-session: it does not patch the running editor. Agents iterating on C++ with the user cannot trigger the editor's own Live Coding compile (Ctrl+Alt+F11 equivalent) or read whether a patch succeeded.

UE 5.8 parity evidence: LiveCodingToolset (1 tool, `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\LiveCodingToolset\`), over `ILiveCodingModule`.

Proposed scope:
- `system.live_coding_compile()` - trigger `ILiveCodingModule::Compile`, run as a job (compile is long-running), returning success/failure + captured log tail.
- `system.live_coding_status()` - enabled?, session started?, last compile result.
- Graceful NOT_AVAILABLE when Live Coding is disabled or unavailable (e.g. -game builds).

Acceptance: with Live Coding enabled, touch a .cpp, trigger via RPC, job completes with patch-applied status matching the editor notification; status RPC reflects it.

## History
- `#1-no-live-coding-rpc` `OPEN` reporter — No in-process Live Coding trigger; run_ubt spawns external UBT and cannot patch the live editor. Epic 5.8 ships LiveCodingToolset over ILiveCodingModule; add compile trigger as job + status readback.
- `#2-live-coding-handler` `IN-REVIEW` developer — Added `Handlers/System/LiveCodingHandler.cpp` with `system.live_coding_compile` + `system.live_coding_status` over `ILiveCodingModule`. Compile is SYNCHRONOUS `Compile(WaitForCompletion, &Result)` (matching Epic's UE 5.8 LiveCodingToolset), NOT the ticket's suggested "job": ILiveCodingModule exposes the compile result ONLY via the synchronous out-param, so a job wrapper could not report success/failure. Captures the LogLiveCoding tail; caches last result in `FPluginState` for the status readback (+ wire-timeout recovery). Graceful `LIVE_CODING_NOT_AVAILABLE` / `LIVE_CODING_NOT_ENABLED`; a real compile failure surfaces as `LIVE_CODING_COMPILE_FAILED` (no fake success). Registration is unconditional; body `__has_include`/`WITH_LIVE_CODING`-guarded (WaterHandler pattern); `Build.cs` soft-links `LiveCoding` under `Target.bWithLiveCoding`. Files: `Handlers/System/LiveCodingHandler.cpp`, `PinWright.Build.cs`, `State/PluginState.h`, `State/PluginState.cpp`. Regression test (adopted red test, flips red→green): `PinWright.system.live_coding.RpcHandlersRegistered`, strengthened with `PinWright.system.live_coding.StatusReportsAvailability` (invokes the real status handler, asserts the availability contract) in `Tests/EditorOps/TestLiveCodingHandlers.cpp`.
