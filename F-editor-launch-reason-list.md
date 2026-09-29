---
id: F-editor-launch-reason-list
title: "List every running Unreal editor on the machine, with its project and the reason PinWright launched it"
status: IN-REVIEW
severity: Medium
category: feature
tags: [proxy, editor-list, editor-start, launch-reason, system-identity, linux, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T00:00:00Z
---

# editor_list with mandatory launch reasons

User requirement: "we must have rpc to list all running unreal editors, even from other repos. and
when running editor we must supply why we run it as a mandatory rpc input", "so we can see all
editors running, and why any of them is running", "and from which project".

Nothing listed editors across checkouts, and nothing recorded why an editor was started: a machine
running several checkouts' editors plus headless workers could only be inspected by hand, and an
agent could not tell its own editor from another stream's.

**Design (user-confirmed):** no registry on disk. Every PinWright launch path appends
`-PinWrightLaunchReason=<reason>` and `-PinWrightLaunchedBy=<editor_start|editor_restart|editor_run_tests|cli>`
to the editor's own command line; `editor_list` reads them back from each process.

## History
- `#1-initial-request` `OPEN` reporter — no census of running editors and no record of why each was launched.
- `#2-reason-and-editor-list` `IN-REVIEW` developer — `reason` required on editor_start/editor_restart/editor_run_tests (`INVALID_REASON`: non-string, blank, >300 chars refused); every launch appends `-PinWrightLaunchReason` (% and " percent-escaped, since FParse::Value has no quote escape) and `-PinWrightLaunchedBy`; new proxy tool `editor_list` enumerates every UnrealEditor* process (Windows CIM query, Linux /proc) and derives project/projectName/checkoutRoot/projectSource, mode, map, logPath, gatewayPort, reason, launchedBy from the command line; `system.identity` gains `launch_reason`/`launched_by` (`Handlers/System/LaunchIdentity.h`, test `PinWright.transport.identity.LaunchSwitchesParseAndDecode`, not yet compiled). OS association launch removed: every editor is a direct detached spawn. Python tests 320 OK; C++ unbuilt.
- `#3-mode-values-in-list` `IN-REVIEW` developer — `editor_list` `mode` is now `visible | offscreen | headless | commandlet | game` (headless on `-NullRHI`; commandlet and game checked first). `editor_test_status` echoes the run's `mode` (live command line, or the log's `Command Line:` line once finished).
