---
id: B-drive-web-handle-collision
title: "drive web observe assigns duplicate pw-N handles within one response"
status: OPEN
severity: Medium
category: bug
tags: [drive, web, cef, observe, handle]
---

# drive web observe assigns duplicate pw-N handles within one response

On `surface:web`, a single `drive.observe` response can carry two different
elements sharing the same handle (`pw-0`). Because `drive.click`/`drive.expect`
re-resolve the target by that handle via
`document.querySelector('[data-pw-id="'+h+'"]')` (`DriveWebBridge.cpp:790`,
which returns the **first** match in document order), a collision silently
routes the action to the wrong element — the agent thinks it clicked the
element it observed, but hits an earlier same-handle element instead.

Root cause is confirmed in the handle minter `hid()` at
`DriveWebBridge.cpp:770-771`: each `BuildQueryElementsJs` run declares a fresh
`var n=0` and mints `pw-'+(n++)` **only for elements without a `data-pw-id`**,
reusing the persistent stamp otherwise. Stamps are intentionally left on the
DOM across observes (`DriveWebBridge.cpp:272-274`). So when a DOM change adds a
new interactable, the next observe reuses old stamps (`pw-0..pw-2`, which do
**not** advance `n`) and then mints `pw-0` again for the new element — a value
already in use. This matches the evidence: after expanding Constructor, БЭКЕНД
(reused `pw-0`) and the new РАСШИРЕННЫЕ НАСТРОЙКИ `<button>` (freshly minted
`pw-0`) collided, while `pw-1`/`pw-2` stayed unique. (Separate from the sibling
finding on omitted custom interactables — this ticket is strictly about handle
uniqueness.)

**Workaround:** re-observe on a fully-settled DOM before acting; if two
elements share a handle, drive by a different unique handle or by coordinates.
**Fix:** make minted ids disjoint from existing stamps — either seed `n` past
the max numeric suffix of existing `[data-pw-id]` at observe start, or hang the
counter on `window` (`window.__pwn`) so it is monotonic across observes instead
of resetting to 0 each call.

## History
- `#1-confirmed-mint-reuse-collision` `OPEN` reporter — Confirmed against source: `hid()` at `DriveWebBridge.cpp:770-771` mints `pw-(n++)` from a per-call `n=0` while reusing persistent `data-pw-id` stamps (`DriveWebBridge.cpp:272-274`); a DOM-added element mints a `pw-N` already held by a reused stamp, and the resolver `querySelector` at line 790 takes the first match. Live PIE evidence: БЭКЕНД div and new РАСШИРЕННЫЕ НАСТРОЙКИ button both returned `pw-0` in one observe.
