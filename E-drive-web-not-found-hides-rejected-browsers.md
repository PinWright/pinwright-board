---
id: E-drive-web-not-found-hides-rejected-browsers
title: "surface=web says 'No live CEF web browser (0 discovered)' when a laid-out UWebBrowser exists but its page never committed - the rejected candidates and the reason are hidden"
status: OPEN
severity: Low
category: ergonomic
tags: [drive, web, cef, observe, discovery, error-message]
encounters: 1
lastSeen: 2026-10-03T13:21:41Z
rice: [1, 2, 1, 1]
priority: 17
---

# surface=web reports "0 discovered" while a visible, sized UWebBrowser exists whose page never committed

`FDriveWebBridge::DiscoverBrowsers()` (`Handlers/Drive/DriveWebBridge.cpp`) silently drops
every `UWebBrowser` that fails `IsDrivableBrowser()`: no cached Slate widget, a URL that is
empty or `about:*` (`IsDrivableUrl`), or zero painted geometry. When nothing survives, every
web verb answers `WEB_BROWSER_NOT_FOUND: No live CEF web browser at browser_index 0
(0 discovered)`. The message reads as "there is no browser", so the caller goes looking for a
missing HUD, when the real state is "a browser is on screen but its document never
committed".

Repro (UE 5.8 Linux, PDS wt2, PinWright ae877ccc, visible editor on :0, no -unattended,
`-nocefaccelpaint`): `editor.play`, `App.Launch DA_Agro_Mission`. The PDS log shows
`WebUI: Loading page: http://pds.local/AgroHUD/index.html` and the scheme handler's
`Assembled HTML index.html`, but CEF delivered no title/console events for ~7 minutes.
During that window:
- `drive.observe {surface:"web"}` -> `WEB_BROWSER_NOT_FOUND ... (0 discovered)`;
- `drive.observe {surface:"game", handle_contains:"WebView"}` listed
  `W_HUD_Agro_WebView_C_0/SWebBrowserView` `visible:true`, 705x464;
- Python: the transient `WebBrowser` reported `get_url() == "about:blank"`, visible True.
After `load_url("http://127.0.0.1:19882/...")` the URL became drivable and the error changed
to `WEB_QUERY_FAILED` (DOM poll timed out), again with no hint that the page had not run any
script yet. All queued CEF events then arrived in one burst (title, console logs) and every
web observe worked. The stall itself is CEF/host-side, not PinWright (it did not reproduce
when the window was re-covered); the PinWright problem is that both errors hide the state
that explains it. Cost: ~15 extra calls (Python object iteration, log greps, X screenshots)
to learn that one browser existed with an uncommitted `about:blank` URL.

**Fix:** when discovery returns no drivable browser but `TObjectIterator<UWebBrowser>` saw
non-CDO candidates, put them in the error's details: per candidate its widget path, URL
(or "empty"), cached-geometry size, and the rejection reason (`no_slate_widget`,
`url_not_committed` for empty/about:*, `zero_geometry`). Say "1 browser found, none
drivable: page not committed (about:blank) - the page is still loading or CEF is not
delivering events; retry, or check the host's WebUI log". For `WEB_QUERY_FAILED`, add the
browser's current URL and title so a page that never ran script is distinguishable from a
JS error. Document both in the drive wiki's web section.

## History
- `#1-initial-report` `OPEN` tester - Filed from the b6 web-observe verification (PDS wt2, PinWright ae877ccc, UE 5.8 Linux, visible editor). `WEB_BROWSER_NOT_FOUND (0 discovered)` was returned while the Agro HUD's UWebBrowser was laid out and visible at 705x464 with `get_url()=="about:blank"`; then `WEB_QUERY_FAILED` once a dev-server URL committed but CEF had not yet delivered script events. Neither error named the candidate or why it was rejected; ~15 extra calls to find out. Raw evidence: scratchpad `b6/web-observe/00-no-browser-before-ready.txt`.
