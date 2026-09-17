---
id: B-editor-dies-silently-cef-pie-lobby
title: "Editor process vanished twice with a CEF-panel sumo lobby live: no Fatal line, no minidump, no crash artifact"
status: OPEN
severity: Medium
category: bug
tags: [editor-stability, cef, webui, pie, no-crash-artifact, needs-evidence, drive]
encounters: 2
lastSeen: 2026-09-16T18:05:35+04:00
---

# Editor died silently twice while a sumo lobby with a live CEF panel was running

Two editor sessions on 2026-09-16 ended mid-frame with **no engine-side crash
path taken**: no `Fatal`, no `Assertion failed`, no `=== Critical error ===`, no
shutdown sequence, and no minidump for either death. The log simply stops.
Both sessions were running a sumo lobby with a live CEF WebUI panel, and both had
logged the accelerated-rendering fallback shortly before.

Scope caveat, stated rather than papered over: this board is nominally MCP-tool
issues only. This is filed here because both deaths happened under PinWright
drive/screenshot traffic and each one destroys the whole MCP session, the same
reason `B-editor-quit-crash-pie-active` and
`E-editor-not-running-cannot-distinguish-crash` live here. Attribution to any MCP
verb is **not** established — see Honest limits.

## Evidence (paths relative to `X:\src\unreal\unreal-fpv\`)

**Death 1 — 13:30:56 UTC.** `Saved/Logs/PDS-backup-2026.09.16-13.30.56.log`
- last line: `[2026.09.16-13.30.56:453][214]LogApp: Received FetchList` +
  a `my-tournaments/invites` payload. Nothing after it.
- `grep -c "Fatal|Assertion failed|=== Critical error"` -> **0**.
- last CEF line, 2.8 s before the death, at **:5080**:
  `[2026.09.16-13.30.53:636][ 47]LogWebBrowser: Error: Accelerated CEF rendering
  selected but OnPaint called. Disabling accelerated rendering for this browser
  window.` (an earlier identical line at **:4815**, 13.26.56:061)
- last PinWright RPC: `editor.console_command` at 13.30.52:578 (**:4903**), ~4 s
  before the death; `editor.screenshot` / `editor.screenshot_window` at 13.29.10 /
  13.29.23.

**Death 2 — 13:39:48 UTC.** `Saved/Logs/PDS-backup-2026.09.16-13.39.48.log`
- last line: `[2026.09.16-13.39.48:372][328]LogApp: RebuildLists: bActive=1
  ViewSource=B_PioneerSumo_C_1 ...`, mid steady-state PIE. Nothing after it.
- `grep -c "Fatal|Assertion failed|=== Critical error"` -> **0**.
- CEF fallback lines at 13.34.21:847 (**:4759**) and 13.34.22:637 (**:4944**),
  ~5 min earlier.
- last PinWright RPC: `editor.screenshot_window`
  (id=71704d8d-415d-9d86-856e-e98150cd89cf) at 13.39.44:927 (**:13140**), 3.4 s
  before the death; and a `drive` web action that timed out at 13.38.24:084
  (**:13048**): `Automation request failed (TIMEOUT): Web action did not apply
  (TIMEOUT)`.
- `[2026.09.16-13.39.47:938] LogAutomationController: Ignoring very large delta of
  2.02 seconds` 0.4 s before the last line — the game thread stalled ~2 s just
  before the process disappeared.

Both backup filenames are the dead log's own last-write time (UE names a rotated
log from its timestamp), so the name and the final line agree on the death moment.

**No `-nocefaccelpaint`.** `LogInit: Command Line:` is **empty** in both logs, so
the editor ran with accelerated CEF painting enabled — the configuration the
memo `cef_accelpaint_automation_crash` says to avoid under automation.

**Correction to the original report: `Saved/Crashes/` is NOT absent.** It holds
four reports from 2026-09-16 — but every one is
`<CrashType>Ensure</CrashType>` / `<IsEnsure>true</IsEnsure>`, and their
timestamps (13:26:49, 13:34:16, 13:37:21, 13:43:19) miss **both** deaths. Ensures
do not terminate the process. So the correct statement is stronger than "no
Crashes folder": the crash handler produced an artifact four times that day and
produced **nothing** for the two deaths, i.e. the process went away without the
engine crash handler running at all. That pattern fits an unhandled fault or a
stack overflow on a **non-engine thread** (CEF has its own), or an external kill,
not a normal UE crash.

(The four ensures are an unrelated PinWright defect —
`Ensure condition failed: ExistingWorld.Equals(ResolvedWorld, ...)`
`[WorldPrecondition.cpp]`, "Handler-defined world 'server' disagrees with resolved
target '/Game/System/FrontEnd/Maps/L_Core.L_Core'". Noted so nobody chases it
here; no ticket filed for it in this pass.)

## Honest limits
- No stack, no minidump, no faulting-thread id for either death. The CEF fallback
  line is **correlation only**: it also appears in sessions that did not die, and
  in death 2 it fired ~5 minutes earlier.
- No MCP verb is implicated. `editor.screenshot_window` ran 3-4 s before each
  death, which is suggestive of nothing on its own — screenshot traffic was
  continuous through both sessions.
- It is not established that this is a PinWright defect at all; it may be an
  engine/CEF issue that PinWright merely lives inside.

## Next step before any code change
1. Rerun the same sumo lobby + CEF panel with `-nocefaccelpaint` on the editor
   command line and drive it the same way. If the deaths stop, the accelerated
   OSR path is the cause and the fix is a launch-argument/default change plus a
   wiki note, not a handler change.
2. If it recurs with the flag set, capture with Windows Error Reporting local
   dumps enabled (`HKCU\Software\Microsoft\Windows\Windows Error Reporting\LocalDumps`)
   so a faulting thread and module exist to read; UE's own handler demonstrably
   does not fire for this death.

severity rationale: impact=editor death, which is the Critical band — but the
board reserves Critical for a crash a tool verb causes, and here neither an MCP
verb nor any mechanism is established, and there is no artifact to act on ×
reach=narrow path (sumo lobby with a live CEF panel) -> Medium. **Re-rate to
Critical the moment a repro pins it to a drive/web verb or a minidump lands.**

## History
- `#1-initial-repro` `OPEN` reporter — Filed: two editor sessions on 2026-09-16
  vanished mid-frame at 13:30:56 and 13:39:48 UTC with a sumo lobby and a live CEF
  panel running, leaving no `Fatal`/assert/critical-error line and no minidump
  (`Saved/Logs/PDS-backup-2026.09.16-13.30.56.log`,
  `Saved/Logs/PDS-backup-2026.09.16-13.39.48.log`; both grep 0 for
  Fatal/Assertion/Critical error). Last CEF line before death 1 was the
  accelerated-rendering fallback at `:5080`; both editors ran with an **empty**
  `LogInit: Command Line:`, i.e. **without** `-nocefaccelpaint`. Correcting the
  original report: `Saved/Crashes/` exists and holds four 2026-09-16 reports, but
  all four are `CrashType=Ensure` (an unrelated PinWright
  `WorldPrecondition.cpp` world-mismatch ensure) at 13:26:49 / 13:34:16 / 13:37:21
  / 13:43:19 — none at either death, so the engine crash handler never ran for the
  deaths themselves. Attribution unproven; first action is a rerun with
  `-nocefaccelpaint`, then WER local dumps if it recurs.
