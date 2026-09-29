---
id: E-identity-no-log-file-path
title: "system.identity does not name the log file the editor writes, so agents guess `Saved/Logs/<Project>.log` and read another editor's log"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [system, identity, logs, console-command, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# system.identity does not name the editor's log file

Several verbs send their useful output to the editor log instead of the response:
`editor.console_command` and `system.console_command` output, `dump*` stat commands, engine warnings
behind a failed call. The wiki told agents to read `Saved/Logs/<ProjectName>.log`. That path is a
guess. When a second editor of the same project is open, the engine finds `<Project>.log` locked and
writes `<Project>_2.log` (`FOutputDeviceFile::CreateWriter`, `OutputDeviceFile.cpp:494-509` on 5.8).
A `-abslog=` launch writes somewhere else entirely. The agent then reads the other editor's log and
draws conclusions from it, which is the same misrouting `system.identity` exists to prevent.

`system.identity` already reports the answering process (pid, instance id, project file, ports) but
not its log file.

Engines: `FGenericPlatformOutputDevices::GetAbsoluteLogFilename()` has the same signature
(`CORE_API static FString`, `GenericPlatformOutputDevices.h:31`) on UE 5.3 through 5.8, and on all
six `OnLogFileOpened` overwrites its cached name with the path the writer actually opened, `_2`
suffix included. No version guard is needed.

**Fix:** `system.identity` returns `log_file`, the absolute path from
`FPlatformOutputDevices::GetAbsoluteLogFilename()` run through `FPaths::ConvertRelativePathToFull`,
read live on each call and omitted when no such file exists on disk.

## History
- `#1-identity-lacks-log-path` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis (plan task 5). `system.identity` names the process but not its log; `editor.console_command` docs pointed at the assumed `Saved/Logs/<ProjectName>.log`, which is the wrong file whenever a second same-project editor holds it. Severity Medium: reading output needs a workaround, and the workaround silently reads the wrong file in the two-editor case.
- `#2-log-file-field-added` `IN-REVIEW` developer - Added `log_file` to `system.identity` in `Source/PinWright/Private/Handlers/System/IdentityHandler.cpp`: `FPaths::ConvertRelativePathToFull(FPlatformOutputDevices::GetAbsoluteLogFilename())`, read live (not cached in `PinWrightEditorIdentity::Measured()`), omitted when the file does not exist; not added to the assertable fields. Named `log_file`, not the plan's `logFile`, because every other field of this response is snake_case (`project_file`, `executable_path`, `bound_http_port`). New test `PinWright.transport.identity.LogFileIsTheOneThisProcessWrites` (`Tests/Transport/TestEditorIdentity.cpp`): field present, absolute, exists, and a freshly logged GUID marker is found in it after `GLog->Flush()`, so a stale sibling log cannot pass. Wiki: `docs/wiki-src/system.md` identity section documents `log_file`; `docs/wiki-src/editor.md` `editor.console_command` output note points at it. Both touched .cpp files compile-checked with `-SingleFile` on 5.8; suite not run.
