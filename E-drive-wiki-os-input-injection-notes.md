---
id: E-drive-wiki-os-input-injection-notes
title: "drive wiki does not say how to fall back to OS-level input injection — the DPI, window-handle and stray-click rules have to be rediscovered, and getting them wrong clicks into another application"
status: OPEN
severity: Low
category: ergonomic
tags: [drive, docs, wiki, input-injection, sendinput, dpi, window-handle, safety]
encounters: 1
lastSeen: 2026-09-09T14:00:00Z
---

# Document the OS-level input fallback on the drive wiki page

When a `drive.*` action verb cannot actuate a widget, the remaining route is an external,
OS-level click (Win32 `SendInput` from PowerShell / a small script) at the element's reported
`geometry.absolute`. `docs/wiki-src/drive.md` describes `geometry.absolute` as "absolute
screen-space" (`drive.md:17`) but says nothing about how to use it from outside the editor, so
each agent rediscovers the same three rules — and the failure mode of the third one is a click
delivered to whatever application happens to be under the cursor.

Three facts, established in a real-click PIE session (UE 5.8, `X:\src\unreal\unreal-fpv-dev`,
plugin `fa755a4f`, 2560x1440 at 150% Windows scaling):

1. **Slate absolute coordinates are physical desktop pixels, 1:1.** A DPI-unaware process sees
   the virtualized logical desktop instead — at 150% scaling PowerShell reported 1707x960 for a
   2560x1440 screen — so every coordinate is off by the scale factor and the click lands
   elsewhere. Any external injector must call `SetProcessDPIAware()` (or run as
   per-monitor-DPI-aware) **before** reading screen metrics or issuing `SendInput`. No scaling
   arithmetic on the drive coordinates is needed once the process is DPI-aware.
2. **`Process.MainWindowHandle` can be 0 for the editor.** Resolving the editor window through
   the .NET property is unreliable; enumerate top-level windows and match the window **class**
   `UnrealWindow` (filtered to the target process id) instead. `drive.list_windows` gives the
   window geometry from the Slate side, but not an HWND.
3. **Guard the injection against a stray target.** In this session one injected click landed in
   the user's terminal because the cursor was over another application. Before pressing, check
   that the window under the cursor position (`WindowFromPoint` -> `GetWindowThreadProcessId`)
   belongs to the intended editor process, and abort otherwise. This is cheap and is the
   difference between a no-op and typing into someone else's window.

Documentation is likely the whole fix here: the notes belong on `docs/wiki-src/drive.md` next to
the action verbs, framed as the escape hatch for when injection through the drive verbs does not
actuate (see `B-drive-click-misses-pie-game-viewport`). If a fixer would rather ship code, the
useful shape is a small guarded helper rather than a new RPC — but the wiki note alone removes
the rediscovery cost and the stray-click hazard.

severity rationale: impact = docs / discoverability friction, with a real but bounded safety
edge (a misdirected click into an unrelated window) x reach = only sessions that fall back to
OS-level injection, a rare path -> Low.

## History
- `#1-os-injection-notes-missing` `OPEN` reporter — Filed from a real-click PIE verification (UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`, 2560x1440 at 150% scaling). Falling back to Win32 `SendInput` at `geometry.absolute` required three undocumented facts: Slate absolute coords are physical pixels 1:1 so the injector must `SetProcessDPIAware()` first (an unaware PowerShell reported 1707x960 for a 2560x1440 desktop); `Process.MainWindowHandle` can be 0 for the editor and the window must be found by the `UnrealWindow` window class; and injected clicks need a `WindowFromPoint` + process-id guard — without it one click in this session landed in the user's terminal. Ask: document all three on `docs/wiki-src/drive.md` beside the action verbs, as the escape hatch referenced from `B-drive-click-misses-pie-game-viewport`.
