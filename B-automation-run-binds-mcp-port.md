---
id: B-automation-run-binds-mcp-port
title: "A plain UnrealEditor-Cmd -ExecCmds automation run starts the PinWright transport and takes the checkout's MCP port, so any test run locks every agent out of editor_start and editor_run_tests"
status: DONE
severity: Medium
category: bug
tags: [transport, port, subsystem, automation, execcmds, editor_start, editor_run_tests, shared-checkout]
encounters: 1
costly: 1
lastSeen: 2026-09-29T00:00:00Z
---

# Automation-only runs bind the MCP port

`UPinWrightSubsystem::Initialize` skips startup only for `IsRunningCommandlet()`
(`Source/PinWright/Private/PinWrightSubsystem.cpp:70-76`). An automation run launched as
`UnrealEditor-Cmd <uproject> -ExecCmds="Automation RunTests <filter>;Quit" ...` is not a commandlet,
so the editor subsystem initializes and binds the HTTP transport on the checkout's derived port
(`:20484` for `X:\src\unreal\unreal-fpv`). A `-game` run has no editor subsystem and does not bind.

Observed 2026-09-29: agents on the PDS checkout ran project (non-PinWright) automation tests this way.
For the whole run the port answered PinWright MCP, so every `editor_start` / `editor_run_tests` from
other sessions was refused with `EDITOR_ALREADY_RUNNING` (`_editor_process_guard`,
`Content/Python/mcp_proxy.py:2439-2478`), naming neither the pid nor that the blocker was a test run
that would exit by itself. MCP-driven work stalled until the test run ended or was killed.

The test run has no use for the transport: nothing drives it over MCP. PinWright's own suite does
need it (it exercises the transport), so the fix must keep that case working.

**Asked for:**
- Do not bind the transport for a run whose command line carries `-ExecCmds="Automation RunTests ..."`
  (or `-ExecCmds` at all) unless explicitly requested, e.g. by an opt-in switch that
  `editor_run_tests` / the capped supervisor adds for PinWright's own suite; or bind an ephemeral
  port for such runs so the checkout's port stays free.
- An explicit opt-out switch (e.g. `-PinWrightNoTransport`) for any launch that must not bind.
- When refusing, report the owner pid, mode (`editor_list` already classifies `commandlet`/`game`/
  `headless`...), launch reason and command line, see `F-multi-editor-per-checkout`.

**Workaround:** wait for the test run to finish, or find and close it via `editor_list`.

## History
- `#1-test-run-holds-mcp-port` `OPEN` reporter - Observed on `X:\src\unreal\unreal-fpv` (UE 5.8): a project automation run (`UnrealEditor-Cmd ... -ExecCmds="Automation RunTests ..."`) bound `:20484` and blocked other sessions' `editor_start` with `EDITOR_ALREADY_RUNNING` for its whole duration; `-game` runs did not. Root cause by source read: the subsystem's only early-out is `IsRunningCommandlet()` (`PinWrightSubsystem.cpp:70`), and the port has no per-launch override (`PinWrightSettings.cpp:120-127`). severity rationale: impact=Medium (soft blocker: wait or kill the other agent's run) x reach=every automation run on a shared checkout, no modifier -> Medium. costly=1 (MCP work stalled behind test runs; editors relaunched).
- `#2-automation-run-leaves-port-free` `IN-REVIEW` developer - `UPinWrightSubsystem::Initialize` now consults `PinWrightTransportLaunchPolicy::SuppressionReason(FCommandLine::Get())` (new header `Source/PinWright/Private/Transport/TransportLaunchPolicy.h`): an editor whose `-ExecCmds` names an `Automation` command does not start the transport (no bind, no `gateway-port` write, no `jobs.jsonl` wipe) and logs why; `-PinWrightTransport` opts back in; `-PinWrightNoTransport` opts any launch out and wins. `-RunningUnattendedScript` alone is NOT a trigger (every `editor_start` offscreen/headless launch carries it). `pinwright_supervisor.suite_argv` adds `-PinWrightTransport`, so `editor_run_tests` and the capped supervisor keep PinWright's own suite on the transport. `editor_start` `extra_args` help names the switch; wiki-src `mcp-transport.md` documents which launches bind. Not done here: richer `EDITOR_ALREADY_RUNNING` refusal (owner pid/mode/reason/command line) - left to `F-multi-editor-per-checkout`; ephemeral-port alternative not needed. Files: `PinWrightSubsystem.cpp`, `Transport/TransportLaunchPolicy.h`, `Tests/Transport/TestTransportLaunchPolicy.cpp`, `Content/Python/pinwright_supervisor.py`, `Content/Python/mcp_proxy.py`, `Content/Python/tests/test_pinwright_supervisor.py`, `docs/wiki-src/mcp-transport.md`. Test: `PinWright.transport.launch_policy.AutomationRunLeavesThePortFree`; Python `tests.test_pinwright_supervisor` (suite argv pins `-PinWrightTransport`). Note: a proxy process started before this change still builds the suite argv without the opt-in until restarted.
- `#3-isolated-child-pin-follows-wipe-guard` `IN-REVIEW` developer - Full-suite run: `PinWright.system.run_tests.IsolatedChildStateBoundary` failed "isolated child skips the shared job-monitor wipe". It is a literal-source pin on `if (!bIsolatedTestChild)` before the jobs.jsonl wipe; the guard is now `if (!bIsolatedTestChild && TransportSuppressedBy.IsEmpty())`, so the isolated-child guarantee still holds (the pin was over-specific, not the behaviour). Kept the extra condition (a no-transport test run must not wipe the live MCP editor's jobs.jsonl) and updated the pinned text in `Tests/EditorOps/TestSystemHandlers.cpp` to the new guard, so reverting either half fails the test; it also now pins the launch-policy wiring in `Initialize`. Same suite log confirms a `-PinWrightTransport` suite run still binds (`HTTP transport binding port 28487`) and `PinWright.transport.launch_policy.AutomationRunLeavesThePortFree` passed.
- `#4-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). `PinWright.transport.launch_policy.AutomationRunLeavesThePortFree` (a plain `-ExecCmds="Automation RunTests ..."` launch suppresses the transport for every separator and casing, `-RunningUnattendedScript` alone does not, `-PinWrightTransport` opts back in, `-PinWrightNoTransport` wins) and `PinWright.system.run_tests.IsolatedChildStateBoundary` (source pin of the policy wiring in `Initialize` and of the jobs.jsonl wipe guard) passed in w23-final. w23-final itself was launched with `-ExecCmds="Automation RunTests PinWright,Quit" -PinWrightTransport` and logged `HTTP transport binding port 28487`, so the opt-in path is proven live. Python `ArgvTest.test_suite_argv_per_mode` (suite argv carries `-PinWrightTransport`) passed. Ask 3 (owner in `EDITOR_ALREADY_RUNNING`) is delivered by `F-multi-editor-per-checkout` (`SlotOwnerTest.*` pass, including `test_run_tests_refusal_carries_the_owner_too`). Limit: no live plain automation run was started to watch the port stay free; that branch is covered by the policy unit test plus the source pin.
