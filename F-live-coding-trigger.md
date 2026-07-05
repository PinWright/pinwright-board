---
id: F-live-coding-trigger
title: "system: in-process Live Coding compile trigger + status readback"
status: OPEN
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
