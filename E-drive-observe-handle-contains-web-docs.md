---
id: E-drive-observe-handle-contains-web-docs
title: "drive.observe handle_contains docs promise a widget/screen-name match, but web handles are flat pw-N stamps, so on surface=web it only substring-matches the opaque id (pw-1 also matches pw-10..pw-19)"
status: OPEN
severity: Low
category: ergonomic
tags: [drive, observe, filter, web, docs]
encounters: 1
lastSeen: 2026-10-03T13:23:32Z
rice: [1, 2, 1, 1]
priority: 17
---

# `handle_contains` on surface=web matches only opaque `pw-N` ids, contrary to the param doc

The `handle_contains` param description (`DriveObserveHandler.cpp`) and `docs/wiki-src/drive.md`
say it matches "the widget-tree path ... e.g. a widget or screen name", and the wiki says
"All of these apply to every surface." On `surface=web` the handle is not a path: the DOM
walk (`FDriveWebBridge::BuildQueryElementsJs`) stamps each element with a flat
`data-pw-id="pw-N"`, so there is no widget, screen or element name to match. The filter
still works mechanically, but only as a substring on the stamp: verified live on the PDS
AgroHUD page, `handle_contains:"PW-1"` returned both `pw-1` and `pw-10`
(`b6/web-observe/04-handle_contains-PW-1.json`). An agent following the doc and passing a
section or element name on web gets an empty list and no explanation.

**Fix (docs, E=1):** in the `handle_contains` param text and the drive.md filter paragraph,
state that on `web` handles are opaque `pw-N` stamps, so `handle_contains` is useful there
only to re-find an already-known handle and is a substring match (`pw-1` matches `pw-10`);
use `label_contains` to find web controls by text. Optional follow-up (not needed now):
add the element's `id`/`class` to the web element so a name filter has something to match.

**Test idea (non-headless, cheap):** `TestDriveWebLive.cpp` already builds a live CEF
fixture window that passes on the X-display drive run; one more latent test there could call
`drive.observe {surface:"web", label_contains:<fixture label in other case>, max_elements:1}`
and assert the one match, covering the web filter path that the offscreen suite skips.

## History
- `#1-initial-report` `OPEN` tester - Found during the b6 web-observe verification (PDS wt2, PinWright ae877ccc, UE 5.8 Linux, visible editor, AgroHUD over pds.local). `label_contains`, `filter` and Cyrillic folding all work on web; `handle_contains:"PW-1"` returned `pw-1` and `pw-10`, and the doc's "widget or screen name" example cannot apply to web's flat `pw-N` handles. Docs-only gap.
