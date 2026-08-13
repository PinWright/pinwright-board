---
id: B-blockout-review-ortho-snap-contradiction
title: "Blockout review misstates orthographic capture behavior"
status: OPEN
severity: Medium
category: bug
tags: [docs, render, camera, orthographic]
encounters: 1
lastSeen: 2026-08-13T00:00:00Z
---

# Blockout review misstates orthographic capture behavior

`blockout-review.md` says every capture surface snaps an arbitrary requested
orthographic rotation to the nearest cardinal axis. That contradicts the typed
`render.capture_open_level` contract and `level-blockout.md`: raw open-level
capture rejects poses more than one degree off an axis with
`UNSUPPORTED_ORTHOGRAPHIC_ROTATION`; only the higher-level camera helpers snap
and report the requested/applied poses.

This sends level authors down the wrong verification path and can make a failed
measurement capture look like a tool regression.

**Fix:** Describe the two behavior families explicitly and add a rendered-doc
contract test so the review and hub pages cannot drift apart again.

## History
- `#1-ortho-contract-contradiction` `OPEN` reporter — Confirmed current `blockout-review.md` says every surface snaps and a tilted request returns the nearest axis, while `render.md`, `level-blockout.md`, and the render-handler tests specify raw-capture rejection with `UNSUPPORTED_ORTHOGRAPHIC_ROTATION`.
