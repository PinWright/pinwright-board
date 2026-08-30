---
id: B-set-window-state-cannot-restore-minimized
title: "editor.set_window_state {state:'restored'} cannot restore a minimized window — the shared selector enumerates only VISIBLE top-level windows, so the one state the verb exists to un-do answers NO_WINDOWS and the only recovery is Win32 outside the RPC surface"
status: OPEN
severity: High
category: bug
tags: [editor, set_window_state, drive, list_windows, window-selector, minimized-window, no-workaround, no-windows, get-all-visible-windows-ordered, editor-chrome, performance-measurement-trap, unobservable-state]
encounters: 1
lastSeen: 2026-08-30T16:06:56+03:00
---

# The one window state the verb exists to fix is the one its resolver cannot see

`editor.set_window_state {state:'restored'}` advertises "un-maximize **AND un-minimize** to a normal
framed window". Against a minimized editor it returns `[NO_WINDOWS] No visible top-level windows are
open` and never reaches `SWindow::Restore()`. The action it would have performed is correct and would
have worked; the resolution step in front of it refuses to hand it the window, because the shared
window selector is built from an enumeration that filters minimized windows out by definition.

There is no in-band recovery. `drive.list_windows` — the discovery verb an agent hits first — returns
`{"windows":[],"count":0}` from the same enumeration, so the agent cannot even *name* the window it
needs to restore. Recovery in this session was Win32 `ShowWindowAsync(hwnd, SW_RESTORE)` against the
editor PID: outside PinWright entirely.

## Measured (13:32 build; the four load-bearing files are identical at HEAD, see below)

The editor was found **already minimized at session start** — not put there by any agent action. In
that state a `CsvProfile FRAMES=150` at a deep-canopy pose on `PW_VegetationTest` reads as a
catastrophic scene regression. Same camera, same scalability, six minutes apart; percentiles below
re-derived by this ticket directly from the CSVs (nearest-rank), not copied from the report:

| window state | FrameTime p10 / p50 / p90 (ms) | GPUTime p10 | `RHI/DrawCalls` p50 | GameThreadTime |
|---|---|---|---|---|
| minimized — `Saved/Profiling/CSV/Profile(20260830_141835).csv` | **333.3300 / 333.3333 / 333.3372** (max 333.5594) | 0.2612 | **0**, on all 150 frames | **39.1727 on all 150 frames** |
| restored — `Profile(20260830_142424).csv` | 17.4984 / 18.1751 / 18.9957 | 16.1537 | 584 (max 638) | 6.68 p50 |

333.33 ms is 1/3 s to five figures across 150 frames — a hard cap, not a cost. `t.MaxFPS` is `0` and
`bThrottleCPUWhenNotForeground` is `False`, so nothing is configured to cap the rate; minimize is a
separate engine path with no cvar. Source report and its surrounding evidence:
`EAContentExamples58/Docs/map/vegetation-performance.md:214-241`, project commit `1f7af8a1`.

**Why this is a PinWright ticket and not a map note:** the profile is not merely low, it is
*internally consistent and wrong in every column at once*. `DrawCalls: 0` and a `GameThreadTime` that
is **bit-identical on all 150 frames** are not measurements, they are a frozen last value being
re-reported. Nothing in the payload is out of range, nothing errors, and no threshold an agent could
set on frame time would distinguish "the forest is 18x too expensive" from "the window is minimized".
This is what turned into an end-user report of *"3 FPS in the forest"* and cost an entire performance
investigation of the scene before the window state was found. The scene actually runs 20.1 ms p50 at
the densest pose (`vegetation-performance.md:249`).

## Mechanism — four links, all re-derived at plugin HEAD `1a9e5778` and engine `C:/UE_5.8`

**1. The engine's enumeration excludes minimized windows, explicitly and unconditionally.**
`C:/UE_5.8/Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp:3789-3799`:

```cpp
void FSlateApplication::GetAllVisibleWindowsOrdered(TArray< TSharedRef<SWindow> >& OutWindows)
{
    for( ... TConstIterator CurrentWindowIt( SlateWindows ); CurrentWindowIt; ++CurrentWindowIt )
    {
        TSharedRef<SWindow> CurrentWindow = *CurrentWindowIt;
        if ( CurrentWindow->IsVisible() && !CurrentWindow->IsWindowMinimized() )   // :3794
        {
            GetAllVisibleChildWindows(OutWindows, CurrentWindow);
        }
    }
}
```

`GetAllVisibleChildWindows` (`:3801-3813`) re-applies the same predicate at `:3803`, so the filter
holds recursively — a minimized parent hides its children too.

**2. PinWright's shared selector is built from exactly that list, and short-circuits when it is
empty.** `Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.cpp`, `ResolveSelectedWindow`
(`:202-297`):

```cpp
FSlateApplication& SlateApp = FSlateApplication::Get();
TArray<TSharedRef<SWindow>> Windows;
SlateApp.GetAllVisibleWindowsOrdered(Windows);                       // :223

if (Windows.Num() == 0)                                              // :225
{
    OutErrorCode = TEXT("NO_WINDOWS");                               // :227
    OutErrorMessage = TEXT("No visible top-level windows are open"); // :228
    return false;
}
```

`FDriveEditorChrome::ResolveWindow` (`:302-312`) is a straight forward to it — the comment at
`:309-310` says so, so that the `editor.*` window verbs "share the exact targeting rule the `drive.*`
surface already uses". Every editor-chrome window verb inherits the exclusion.

**3. `editor.set_window_state` resolves through it before it acts.**
`Source/PinWright/Private/Handlers/Editor/EditorWindowHandlers.cpp`, registered at `:772`, summary
`:773-777` (the advertised un-minimize is `:774`), body:

```cpp
if (!FDriveEditorChrome::ResolveWindow(                                    // :828
        FDriveHandlerCommon::ParseWindowSelector(Ctx), Window, WindowTitle, ErrorCode, ErrorMessage))
{
    Ctx.SendError(ErrorCode, ErrorMessage);                                // :831
    return true;
}
...
case EDesiredWindowState::Restored:  Window->Restore();  break;            // :840
```

`:840` is the whole capability, and `:831` is where a minimized target dies. Nothing between them can
run.

**4. The action would have worked, and it is the same Win32 call the out-of-band recovery made.**
`SWindow::Restore()` — `C:/UE_5.8/Engine/Source/Runtime/SlateCore/Private/Widgets/SWindow.cpp:1762-1768`
— forwards to `NativeWindow->Restore()`, and `FWindowsWindow::Restore()`
(`Engine/Source/Runtime/ApplicationCore/Private/Windows/WindowsWindow.cpp:712-720`) is
`::ShowWindow(HWnd, SW_RESTORE)` at `:718`. The recovery that worked reached *past* PinWright to make
the same call PinWright's own handler would have made a frame later. **This bounds the fix: no new
capability is needed, only a resolution path that can name a minimized window.**

## Both minimized readbacks are structurally unreachable on every path a caller can aim

`EditorWindowHandlers.cpp:836` (`bWasMinimized`) and `:848` (`bIsMinimized`) are the **only two**
`IsWindowMinimized()` reads in the entire plugin (`grep -rn IsWindowMinimized Source/` returns them
plus three comments). Both sit downstream of the resolver:

- **`window_index`** (`:232-243`) indexes into `Windows` → every element passed `!IsWindowMinimized()`.
- **`window_title`** (`:244-261`) iterates `Windows` → same. Its error text even says *"No **visible**
  window title contains '%s'"* (`:258`).
- **Neither set** (`:262-284`) takes `SlateApp.GetActiveTopLevelWindow()` at `:264`, which is
  `return ActiveTopLevelWindow.Pin();` (`SlateApplication.cpp:2935-2938`) — an activation-tracked weak
  pointer with **no** visibility filter. This is the one route by which a minimized `SWindow` could
  reach `:840`, and it is unusable: it cannot be aimed at a chosen window, and it is gated behind the
  `Windows.Num() == 0` check at `:225`, so it survives only while some *other* window is still visible.

So `wasMinimized` / `isMinimized` — the vocabulary the response shape already carries — can publish
`true` only by accident, on the one path the caller cannot request. In the observed condition (the
whole editor minimized) `Windows` is empty, `:225` fires, and they are never computed at all.

## The plugin reasoned about this exclusion twice and stopped one step short each time

This is the strongest fact in the ticket: the mechanism is **already written down in the source**, for
a sibling verb, with the wrong conclusion drawn about its blast radius.

`Source/PinWright/Private/Handlers/Drive/DriveListWindowsHandler.cpp:21-23`:

> *"(No minimized flag: list_windows enumerates via GetAllVisibleWindowsOrdered, which excludes
> minimized windows, so a minimized window never appears here and the field could only ever read
> false.)"*

Repeated at `DriveEditorChrome.cpp:393-395` and, at greatest length, at `DriveEditorChrome.h:40-45`,
which goes as far as naming the shared consumer:

> *"…surfacing minimized state would require broadening that shared enumeration, which shifts the
> window_index targeting order for every editor-chrome drive verb (its own ticket)."*

Both notes frame the consequence as **reporting** — a field that would always read `false` — and the
cost of change as **index ordering**. Neither notices that the same enumeration is what
`editor.set_window_state` resolves through, so the exclusion does not merely make a flag boring: it
makes a *documented control capability* unexecutable. The reasoning was correct and its scope was one
verb too narrow.

**And the wiki turned that gap into a false escape hatch.** `Docs/wiki-src/drive.md:106` closes the
`list_windows` note with: *"to observe a specific window's minimized state use
`editor.set_window_state`'s readback."* That sentence points the reader at the one verb that cannot
resolve a minimized window. `Docs/wiki-src/editor.md:192` likewise still promises `'restored'` will
"un-maximize **and** un-minimize". Both need correcting whatever else is done here.

## Fix

**1. A minimized-tolerant resolution path, added as a *fallback* so nothing re-orders.** The objection
that blocked this before (`B-list-windows-maximized-geometry` `#3`: broadening the enumeration
"shifts the `window_index` targeting order for every editor-chrome drive verb") is answered by
consulting the wider list **only when the visible enumeration yields no match** — indices over visible
windows keep their exact current values, and minimized windows occupy positions after them. The
engine already exposes the unfiltered list publicly and cheaply:

- `FSlateApplication::GetTopLevelWindows()` — `SlateApplication.h:1688`, inline
  `{ return SlateWindows; }`. No filter of any kind. Top-level only (it does not recurse child
  windows, which `GetAllVisibleWindowsOrdered` does), which is exactly right for a fallback: a
  minimized *top-level* window is the case that matters.
- `FSlateApplication::GetInteractiveTopLevelWindows()` (`SlateApplication.h:1492`,
  `SlateApplication.cpp:3762-3787`) also returns `SlateWindows` verbatim (`:3785`) when no modal window
  is up, but is modal-dependent and therefore the weaker choice.
- Resolving the native `HWND` (`SWindow::GetNativeWindow()`, `SWindow.h:586` →
  `FWindowsWindow::GetHWnd()`, `WindowsWindow.h:45`) and calling `ShowWindowAsync` is a third option,
  but it is unnecessary: once the `SWindow` is in hand, `Window->Restore()` at
  `EditorWindowHandlers.cpp:840` already reaches `::ShowWindow(HWnd, SW_RESTORE)`.

Scope the fallback narrowly if the shared resolver is felt to be too load-bearing: `set_window_state`
could take a `minimized-tolerant` resolution of its own rather than changing `ResolveSelectedWindow`
for all callers. Either way the `NO_WINDOWS` message should stop being the terminal answer for a verb
whose registered summary promises un-minimize.

**2. Make the state observable before it is trusted.** An agent must be able to *detect* a minimized
editor without inferring it from an empty array:

- Restore the `minimized` field `B-list-windows-maximized-geometry` `#3` removed, on records the new
  fallback enumeration can now produce — with those records appended after the visible ones so
  existing indices are stable. The field stops being "could only ever read false" the moment the
  fallback exists; the two halves of this fix unlock each other.
- Better still for the failure that actually cost the investigation: a `mainWindowMinimized` (or
  equivalent) flag on the profiling verbs, whose numbers are the ones the state poisons —
  `performance.start_profiling` / `performance.stop_profiling` / `performance.run_benchmark`
  (`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp:252` / `:283` / `:860`). A profile
  captured against a minimized window should say so in its own response, because nothing in its
  numbers ever will.

**3. Docs.** `Docs/wiki-src/drive.md:106` must stop redirecting minimized-state observation to
`editor.set_window_state`, and `Docs/wiki-src/editor.md:192` must either become true or say the
un-minimize half currently only applies to a window that is minimized *while another window is
visible*.

**Workaround until then:** none inside PinWright. Win32 against the editor PID — `IsIconic(hwnd)` to
detect, `ShowWindowAsync(hwnd, SW_RESTORE)` to recover. `drive.list_windows` returning
`{"windows":[],"count":0}` on a live editor is the only in-band tell, and it is indistinguishable in
shape from a Slate teardown.

## Same shape as — the class applies to the discovery verb, not the control verb

The session's recurring class, stated on `B-foliage-paint-does-no-ground-projection` § *Same shape
as*: *the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported.*

`editor.set_window_state` is **not** a member — it errors honestly, and `NO_WINDOWS` is a true
statement about the list it was given. `drive.list_windows` **is** a textbook member: it returns
`{"windows":[],"count":0}`, both numbers correct, from a live editor with an open main window, and the
deciding fact (a window exists and is minimized) is one the record shape was deliberately built to
never carry. The profile CSV is the same class again at one remove — `DrawCalls: 0` and a frozen
`GameThreadTime` are correct readings of a window that is not rendering, presented as a scene cost.

That is what makes the pair expensive: an honest error on the control verb and a silent-correct empty
answer on the discovery verb combine into a state that is neither fixable nor detectable in-band.

## Dedup

Board searched for `minimiz`, `NO_WINDOWS`, `GetAllVisibleWindowsOrdered`, `window_state`, `333`,
`ShowWindow`, `IsIconic`, `SW_RESTORE`, `MaxFPS`, and every filename containing `window`.
**No ticket mentions `NO_WINDOWS` or a minimized window that cannot be selected.** The four neighbours:

- **`B-list-windows-maximized-geometry`** (IN-REVIEW, Medium) — its *title* still reads "omits
  per-window maximize/**minimize** state", but its `#3-test-phase-fix` **deliberately removed** the
  `minimized` field it had shipped in `#2`, on the grounds that it "could only ever read `false`" and
  that broadening the enumeration "belongs in its own ticket". So it does **not** own the field ask —
  it disowned it and nominated this ticket. Filed here as one ticket covering both halves, because the
  selector half and the field half are the same change: the field is tautological until the resolver
  can see a minimized window, and the resolver fix makes the field meaningful for free. That ticket's
  own defect (a maximized window reading as plain) is untouched and stays fixed. Cross-referenced
  there, per the `B-pcg-connect-pins-silently-replaces-edge` `#3-namespace-transaction-gap-now-filed`
  precedent for closing out a carve-out.
- **`E-resize-window-maximized-no-restore`** (IN-REVIEW, Medium) — the ticket that *created*
  `editor.set_window_state` (`#2-go`). Same verb, one state over: it fixed the maximized dead-end and
  the minimized one was never exercised, because a maximized window resolves fine. Not a duplicate —
  its subject is that no window-state verb existed at all; this one's is that the verb that now exists
  cannot reach one of its three states.
- **`B-drive-window-selector-param-unreachable`** (IN-REVIEW, Medium) — also a selector defect, but at
  the dispatcher's param allowlist: `drive.*` verbs reject `window_index`/`window_title` as
  `UNKNOWN_PARAMS` before the handler runs. Different layer (param spec vs enumeration), different
  verbs (`drive.*` action verbs, not `editor.set_window_state`, which declares
  `DRIVE_WINDOW_SELECTOR_PARAMS` at `EditorWindowHandlers.cpp:779` and does reach its handler).
- **`B-frame-graph-tabbed-bp-window-untargetable`** (WONTFIX, Medium) — the other "selector cannot
  reach the window I want" ticket. Different cause (several BP editors share one `SWindow`, title
  substring cannot disambiguate tabs) and it has a clean in-band workaround, which is why it was
  closed; this one has none.

Nothing on the board covers the performance-measurement trap either: `B-performance-run-benchmark-no-completion-signal`
and `F-console-batch-get-cvar-values` matched the `MaxFPS`/`333` greps on unrelated text.

## Severity

**High.** Impact class is the rubric's *"hard blocker with no workaround (a stub, a missing verb, or
rejecting valid input), so a reasonable task is impossible"* band, at its top: the verb rejects a
valid, documented request (`state:'restored'` against a minimized window — precisely the capability
`:774` advertises), the task it blocks is *restoring the editor*, and the recovery is not a workaround
within the tool but a Win32 call against the process from outside it. The blocker compounds rather
than sitting alone — the same enumeration makes the state unobservable through every verb in the
plugin, so an agent in this state cannot detect it, cannot fix it, and receives a profile whose every
column is internally consistent and wrong.

**Critical argued and declined.** The Critical band is *"editor crash, or a write that corrupts or
loses asset data."* Nothing crashes and nothing is written: the harm is a corrupted *conclusion* (an
18x false performance reading, and the wasted investigation behind it), and the underlying state is
fully reversible from outside the tool within seconds once identified. A conclusion is not asset data.
Declined on that basis; what would flip it is evidence of a write taken on the strength of the false
reading — a scene edited or a setting committed to fix a regression that was never there. None
occurred here.

**Medium argued and declined.** Medium is *"soft blocker. Doable, but only via a documented
workaround, a source dive, or many extra calls."* Every one of those escapes stays inside the tool
surface, and every one of them fails here: there are no extra calls to make (the discovery verb
reports an empty list), and a source dive tells you only that the path does not exist. The recovery
that worked required leaving the RPC surface for the OS. That is the hard-blocker band, not the soft
one.

**Reach modifier declined in both directions, and named.** No bump **down** to Medium as a rare edge
path: the state is entered *by accident* — the editor was found minimized at session start, before any
agent action — so exposure is not gated on an agent choosing to call a rare verb; and the resolver is
shared, so in that state every editor-chrome `drive.*` and `editor.*` window verb answers `NO_WINDOWS`
together (`ResolveWindow`, `DriveEditorChrome.cpp:302-312`), not `set_window_state` alone. No bump
**up**: window-state control is honestly not an every-session verb — a session whose editor is normal
never calls it — and a bump up from High lands on Critical, refused above on its merits. High stands
unmodified.

severity rationale: impact=hard blocker with no in-band workaround on a documented capability, plus
the blocking state is unobservable through every verb so it manufactures a false catastrophic
performance reading x reach=not an every-session verb, but entered by accident and shared by every
editor-chrome window verb (declined both ways) -> High.

## Provenance of the numbers

Measured on the **13:32 editor build**, reported as carrying plugin commit `d8f1bc32`; the
build↔commit mapping is the session's record, not re-verified against the loaded binary here — and it
is immaterial, because `git diff d8f1bc32..1a9e5778` over the four load-bearing files
(`DriveEditorChrome.cpp`, `DriveEditorChrome.h`, `EditorWindowHandlers.cpp`,
`DriveListWindowsHandler.cpp`) is **empty**, and all four are clean in the working tree. The measured
behaviour therefore applies to HEAD verbatim. Every plugin and engine `file:line` above was opened and
re-read at HEAD `1a9e5778` / `C:/UE_5.8` for this ticket; percentiles were recomputed from the two
CSVs rather than copied (they differ from the report in the last digit or two — 17.4984 vs 17.56 p10,
6.68 vs 6.37 GameThreadTime — a percentile-convention/statistic difference that changes nothing).

Two stale citations noticed while re-deriving, both pre-existing and neither worth a ticket:
`B-list-windows-maximized-geometry` `#3` cites `SlateApplication.cpp:3742/3751` for
`GetAllVisibleWindowsOrdered`, which lands on `ProcessCursorReply` and a blank line in `C:/UE_5.8`
(the function is `:3789`, the filter `:3794`) — most likely a different engine install; and it cites
`DriveEditorChrome.cpp:222` for the enumeration call, which is `:223` at HEAD.
`B-frame-graph-tabbed-bp-window-untargetable` cites `:245` for the title `Contains` line, now `:248`.

## History
- `#1-selector-cannot-see-a-minimized-window` `OPEN` reporter — `editor.set_window_state {state:"restored"}` returns `[NO_WINDOWS] No visible top-level windows are open` against a minimized editor and never reaches `SWindow::Restore()`, so the one state the verb's own registered summary promises to un-do (`EditorWindowHandlers.cpp:774`, "un-maximize AND un-minimize") is the one it cannot target. Mechanism re-derived across four links at plugin HEAD `1a9e5778` and engine `C:/UE_5.8`: `FSlateApplication::GetAllVisibleWindowsOrdered` (`SlateApplication.cpp:3789-3799`) filters `IsVisible() && !IsWindowMinimized()` at `:3794`, re-applied recursively to child windows at `:3803`; PinWright's shared `ResolveSelectedWindow` (`DriveEditorChrome.cpp:202-297`) builds its candidate list from exactly that call at `:223` and returns `NO_WINDOWS` at `:225-229` when it is empty; `FDriveEditorChrome::ResolveWindow` (`:302-312`) forwards straight to it so every editor-chrome window verb inherits the exclusion; and `editor.set_window_state` (registered `:772`) resolves through it at `:828` and errors out at `:831`, so `Window->Restore()` at `:840` is unreachable. Established that the ACTION is not the defect: `SWindow::Restore()` (`SWindow.cpp:1762-1768`) forwards to `FWindowsWindow::Restore()` (`WindowsWindow.cpp:712-720`) which is `::ShowWindow(HWnd, SW_RESTORE)` at `:718` — literally the same Win32 call the out-of-band recovery (`ShowWindowAsync(hwnd, SW_RESTORE)` against the editor PID) had to make from outside PinWright, which bounds the fix to a resolution path and no new capability. Established that BOTH minimized readbacks are structurally unreachable on any path a caller can aim: `EditorWindowHandlers.cpp:836`/`:848` are the only two `IsWindowMinimized()` reads in the plugin, and the `window_index` (`:232-243`) and `window_title` (`:244-261`) branches both index into the filtered list, while the no-selector branch's `GetActiveTopLevelWindow()` (`:264`; `SlateApplication.cpp:2935-2938`, an unfiltered weak pointer) is the sole route a minimized window could take and is both unaimable and gated behind the `Windows.Num() == 0` check at `:225`. Strongest fact: the plugin ALREADY reasoned about this exclusion twice and drew the wrong scope — `DriveListWindowsHandler.cpp:21-23`, `DriveEditorChrome.cpp:393-395` and at greatest length `DriveEditorChrome.h:40-45` name the filter, name the shared `window_index` consumer, and conclude only that a `minimized` field "could only ever read false" and that broadening the enumeration is "its own ticket"; none notices that the same enumeration makes a documented CONTROL capability unexecutable. `Docs/wiki-src/drive.md:106` compounds it by redirecting minimized-state observation to `editor.set_window_state`'s readback — the exact verb that cannot resolve a minimized window — and `Docs/wiki-src/editor.md:192` still promises the un-minimize half. Cost measured and re-derived from the CSVs by this ticket rather than relayed: minimized (`Saved/Profiling/CSV/Profile(20260830_141835).csv`) FrameTime p10/p50/p90 = 333.3300/333.3333/333.3372 ms with `RHI/DrawCalls` 0 on all 150 frames and `GameThreadTime` bit-identical at 39.1727 on all 150; restored (`Profile(20260830_142424).csv`) 17.4984/18.1751/18.9957 with 584 draws — a hard 1/3 s cap (`t.MaxFPS` 0, `bThrottleCPUWhenNotForeground` False, no cvar), reading as an ~18x scene regression with no column out of range and no threshold able to distinguish it; that is what produced an end-user "3 FPS in the forest" report and cost a whole performance investigation, against an actual 20.1 ms p50 at the densest pose (`EAContentExamples58/Docs/map/vegetation-performance.md:214-241,249`, project commit `1f7af8a1`). The editor was found minimized at session start, so the state is entered by accident, not by an agent action. Fix asked in three parts: a minimized-tolerant resolution added as a FALLBACK consulted only when the visible enumeration yields no match — which answers verbatim the objection `B-list-windows-maximized-geometry` `#3` used to decline broadening (index order over visible windows is unchanged) — using the engine's public unfiltered `FSlateApplication::GetTopLevelWindows()` (`SlateApplication.h:1688`, inline `return SlateWindows;`) in preference to the modal-dependent `GetInteractiveTopLevelWindows()` (`:1492`, `.cpp:3762-3787`, returns `SlateWindows` verbatim at `:3785`) or an `HWND` route (`SWindow.h:586`, `WindowsWindow.h:45`) that the existing `Restore()` already makes unnecessary; restoring the `minimized` field that `#3` removed, on the records the fallback newly produces and appended after the visible ones so indices stay stable; and a `mainWindowMinimized`-style flag on the profiling verbs whose numbers the state poisons (`PerformanceHandler.cpp:252`/`:283`/`:860`), because nothing in a profile's own numbers will ever say so. Cross-linked into the session's recurring class with the distinction stated rather than restated: `set_window_state` is NOT a member (it errors honestly and `NO_WINDOWS` is true of the list it was given), but `drive.list_windows` is a textbook member — `{"windows":[],"count":0}`, both numbers correct, from a live editor, with the deciding fact one the record shape was built never to carry — and the pair is what makes the state neither fixable nor detectable in-band. Dedup: searched `minimiz`, `NO_WINDOWS`, `GetAllVisibleWindowsOrdered`, `window_state`, `333`, `ShowWindow`, `IsIconic`, `SW_RESTORE`, `MaxFPS` and every `window` filename; nothing on the board mentions `NO_WINDOWS` or an unselectable minimized window. Filed as ONE ticket covering both the selector half and the field half, because `B-list-windows-maximized-geometry` `#3` explicitly removed the `minimized` field and nominated "its own ticket" for the enumeration change, and because the two halves unlock each other (the field is tautological until the resolver can see a minimized window). `E-resize-window-maximized-no-restore` created this verb and never exercised the minimized state; `B-drive-window-selector-param-unreachable` is the param-allowlist layer, not the enumeration (`set_window_state` declares `DRIVE_WINDOW_SELECTOR_PARAMS` at `:779` and does reach its handler); `B-frame-graph-tabbed-bp-window-untargetable` is WONTFIX with a clean in-band workaround, which this has none of. Severity High: hard-blocker band at its top (a valid documented request rejected, with the only recovery outside the RPC surface), with Critical declined because nothing crashes and no asset data is written or lost — the corrupted artefact is a conclusion, reversible in seconds once identified — and Medium declined because every escape the Medium band names (documented workaround, source dive, extra calls) stays inside the tool surface and all three fail here. Reach declined in both directions: no bump down because the state is entered by accident and the shared resolver takes every editor-chrome window verb down with it (`DriveEditorChrome.cpp:302-312`), no bump up because window-state control is genuinely not an every-session verb and a bump from High lands on the Critical already refused. Provenance: measured on the 13:32 build (reported as plugin commit `d8f1bc32`; mapping not re-verified against the binary and immaterial, since `git diff d8f1bc32..1a9e5778` over all four load-bearing files is empty and all four are clean in the working tree, so the behaviour applies to HEAD verbatim). Noted while re-deriving, not filed: `B-list-windows-maximized-geometry` `#3`'s engine citation `SlateApplication.cpp:3742/3751` does not land in `C:/UE_5.8` (`ProcessCursorReply` and a blank line; the function is `:3789`/`:3794`) and its `DriveEditorChrome.cpp:222` is `:223` at HEAD, and `B-frame-graph-tabbed-bp-window-untargetable`'s `:245` is now `:248`.
