---
id: B-drive-browser-index-param-unreachable
title: "drive web browser selector (browser_index) rejected as UNKNOWN_PARAMS despite being documented + implemented"
status: OPEN
severity: Medium
category: bug
tags: [drive, web, cef, param-spec, unreachable-code, doc-mismatch]
encounters: 1
lastSeen: 2026-09-16T18:05:35+04:00
---

# drive `browser_index` rejected as UNKNOWN_PARAMS on every web verb

## What's wrong
Every `surface=web` drive verb reads `browser_index` in its handler body, but no
verb declares it in its `RPC_PARAMS`, so the dispatcher's param allowlist rejects
the payload with `UNKNOWN_PARAMS` before the handler runs. The browser-selection
code is dead: only the browser `SelectBrowser(0)` happens to return is reachable.

**The parameter parser is wrong, not the docs.** The docs match the handler
behaviour; the declaration is the missing half. See Guilty source below.

This is the exact sibling of `B-drive-window-selector-param-unreachable`
(editor-chrome `window_title`/`window_index`, IN-REVIEW). That fix declared the
four window-selector keys at the three drive declaration sites and did **not**
cover `browser_index` — the same class, one selector left behind.

## Observed consequence
With a host PIE instance and a client PIE instance both running a CEF HUD, only
browser 0 is reachable, and *which* browser that is is not stable: `SelectBrowser`
picks from `DiscoverBrowsers()`, which sorts live browsers by **on-screen area,
descending** (`DriveWebBridge.cpp:334-341`), so the index a browser occupies
changes as browsers are created, destroyed or resized. The verifier had to close
the client's CEF panel to make the host panel become browser 0 before it could be
driven. `root_index` does not substitute: it is the live-UMG root selector and is
never consulted on the web path.

## Verbatim repro (from `Saved/Logs/PDS-backup-2026.09.16-13.30.56.log:4837`)
```
call("drive.observe", { surface: "web", browser_index: 1 })
->
[UNKNOWN_PARAMS] Unknown parameter(s) for 'drive.observe': [browser_index].
Valid parameters: [surface, instance_name, instanceName, root_index, rootIndex,
window_title, title, window_index, index, screenshot, screenshot_mode, mark_cap,
interactables_only, max_elements, max_bytes, include_journal, journal_since].
Call 'drive.observe' with no 'args' field to fetch its wiki page.
```
The "Valid parameters" set never contains `browser_index` on any drive verb, so
documented multi-browser targeting is impossible.

## Guilty source (ground truth, read verbatim, paths relative to `X:\src\unreal\unreal-fpv\`)
Declared nowhere — the three drive declaration sites plus the two standalone verbs:
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveObserveHandler.cpp:15-30`
  — `drive.observe`'s `RPC_PARAMS(...)` block. Declares `surface`,
  `instance_name`/`instanceName`, `root_index`/`rootIndex`,
  `DRIVE_WINDOW_SELECTOR_PARAMS`, `screenshot`, `screenshot_mode`, `mark_cap`,
  `interactables_only`, `max_elements`, `max_bytes`, `include_journal`,
  `journal_since`. **No `browser_index`.**
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveActionHandlers.cpp:44-58`
  — the shared `DRIVE_COMMON_ACTION_PARAMS` macro feeding the action verbs
  (`drive.click` at :136, and hover/scroll/type/key/drag). **No `browser_index`.**
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveExpectHandler.cpp:17`
  and `.../DriveWaitHandler.cpp:21` — same omission (`grep browser_index` over
  `Handlers/Drive/*Handler*.cpp` returns zero hits in any `RPC_PARAM_*` line).

Read by every web handler body:
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveWebHandlers.cpp`
  — `const int32 BrowserIndex = Ctx.GetInt(TEXT("browser_index"), 0);` at
  **:330** (`ObserveWeb`), **:412** (`ExpectWeb`), **:460** (`ClickWeb`),
  **:492** (`TypeWeb`), **:523** (`ScrollWeb`), **:560** (`HoverWeb`),
  **:596** (`KeyWeb`), **:650** (`DragWeb`), **:685** (`WaitForWeb`).
- `.../DriveWebHandlers.cpp:91` — the error text itself names the parameter:
  `"No live CEF web browser at browser_index %d."`
- `.../DriveWebHandlers.h:31` — the class contract documents it:
  `"surface=web drive.observe: select the browser (browser_index, default 0)"`.
- `.../DriveWebBridge.cpp:346-350` — `SelectBrowser(Index)` indexes
  `DiscoverBrowsers()`, whose ordering is the area sort at `:334-341`.

The rejecting guard:
- `Plugins/PinWright/Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:176-196`
  — `ValidateHandlerParams` builds `KnownParams` from the verb's declared
  `RPC_PARAMS` only, and emits `UNKNOWN_PARAMS` for any other top-level key.

Documented in the wiki (both the generated tree and its hand-authored source):
- `Saved/PinWright/wiki/drive.observe.md:32` — "…`window_title`/`window_index`
  (editor chrome), `browser_index` (web)…". The **auto-generated `**Parameters**`
  list on the same page (lines 11-27) omits `browser_index`**, so the page
  contradicts itself — the same tell the window-selector ticket recorded.
- `Saved/PinWright/wiki/drive.md:15` and `:62` — "Selected by `browser_index`
  (default 0)"; "Web targets a browser by `browser_index` (default 0) instead of
  `instance_name`/`root_index`".
- Overlay source: `Plugins/PinWright/docs/wiki-src/drive.md:11`, `:58`, `:68`,
  `:106`, `:110` (never hand-edit the generated tree).

Already measured by the project's own guard, and still exempted:
- `Plugins/PinWright/Source/PinWright/Private/Tests/Infra/TestDeclaredParamCoverage.cpp:1866-1892`
  — the `KnownUndeclaredHelperReads()` baseline carries
  `drive.click:browser_index`, `drive.drag:…`, `drive.expect:…`, `drive.hover:…`,
  `drive.key:…`, `drive.observe:…`, `drive.scroll:…`, `drive.type:…`,
  `drive.wait_for:browser_index` — nine entries, one per web verb. The list's own
  header (`:1684-1690`) states no entry is endorsed and that the fix is to declare
  the key and delete the line.

## What it should do
`drive.observe` / `drive.expect` / `drive.wait_for` / the six action verbs should
accept `browser_index` (integer, default 0) when `surface=web`, resolving to the
browser `SelectBrowser` indexes. Then the two PIE instances in a host+client sumo
session are independently drivable without closing one of them.

**Fix:** declare `RPC_PARAM_OPT("browser_index", "integer", ...)` at the four
declaration sites — `DriveObserveHandler.cpp:15-30`, the
`DRIVE_COMMON_ACTION_PARAMS` macro at `DriveActionHandlers.cpp:44`,
`DriveExpectHandler.cpp:17` and `DriveWaitHandler.cpp:21` — mirroring how
`B-drive-window-selector-param-unreachable` declared the window keys. No
handler-body change is needed: every `*Web` entry point already reads the key.
Then **delete the nine `drive.*:browser_index` lines from
`TestDeclaredParamCoverage.cpp:1866-1892`** (the baseline is a ratchet; a stale
entry is reported as a failure). A regression test alongside
`Tests/Drive/TestDriveWindowSelectorParams.cpp` should assert all nine web verbs
declare the key and that `{surface:"web", browser_index:1}` is no longer refused
`UNKNOWN_PARAMS` on the wire.

Worth considering in the same pass, but **not** a substitute for the fix: index 0
is defined by a mutable area sort, so a stable handle (browser URL/title
substring, à la `window_title`) would survive browsers appearing and disappearing.
File separately if wanted; declaring `browser_index` is the minimal correct fix.

severity rationale: impact=hard blocker with no workaround inside the tool (a
second live CEF panel is unreachable; the only way through is to destroy one
browser, which changes the state under test) × reach=web drive verbs are a
sub-path, but the host+client PIE pair is the normal shape of every sumo/WebUI
session here -> Medium. Matches the sibling `B-drive-window-selector-param-unreachable`.

## History
- `#1-initial-repro` `OPEN` reporter — Filed: every `surface=web` drive verb
  rejects the documented `browser_index` selector with `UNKNOWN_PARAMS` because no
  verb declares it in `RPC_PARAMS`, even though all nine `*Web` handler bodies read
  it (`DriveWebHandlers.cpp:330/412/460/492/523/560/596/650/685`), the header
  contract documents it (`DriveWebHandlers.h:31`), and the wiki documents it
  (`Saved/PinWright/wiki/drive.md:15,62`, `drive.observe.md:32`; source
  `docs/wiki-src/drive.md:11,58,68,106,110`). **The parser is wrong, not the
  docs.** Live refusal captured in `Saved/Logs/PDS-backup-2026.09.16-13.30.56.log:4837`.
  `root_index` is not a substitute — it is the live-UMG root selector and is never
  read on the web path. Consequence observed: with a host and a client PIE instance
  each running a CEF HUD, only browser 0 is reachable and its identity shifts with
  the on-screen-area sort in `DriveWebBridge.cpp:334-341`, so the verifier had to
  close the client panel to drive the host panel. The project's own guard already
  measured all nine pairs into the `KnownUndeclaredHelperReads()` baseline at
  `TestDeclaredParamCoverage.cpp:1866-1892`, where they remain exempted. Same class
  as `B-drive-window-selector-param-unreachable`, whose fix covered the
  editor-chrome window keys and left the web browser key behind.
