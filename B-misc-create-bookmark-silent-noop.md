---
id: B-misc-create-bookmark-silent-noop
title: "misc.create_bookmark is a silent no-op that echoes its input (never reads the camera, never stores a bookmark)"
status: IN-REVIEW
severity: High
category: bug
tags: [misc, bookmark, viewport, camera, silent-success, no-op]
---

# misc.create_bookmark is a silent no-op that echoes its input (never reads the camera, never stores a bookmark)

`misc.create_bookmark` advertises (handler description + auto-generated wiki):
*"Save the current viewport camera transform to a bookmark slot."* But the
handler **never reads the viewport camera and never writes any bookmark**. It
just reads the optional `location`/`rotation` params (defaulting to
`ZeroVector`/`ZeroRotator` when absent), logs one `UE_LOG` line, and echoes
`index`/`name`/`location`/`rotation` straight back as success
(`MiscHandler.cpp:311-360`). There is no `GCurrentLevelEditingViewportClient`
/ `LevelViewportClient` read, no `IBookmarkTypeTools`/`UBookmark` write, no
world bookmark-array mutation anywhere in the body.

Called the natural, documented way — `{index, name}` with no
`location`/`rotation`, expecting it to capture the *current* framing — it
returns a confident success reporting the saved transform as
**origin `{0,0,0}` / `{0,0,0}`**, which is neither the live camera nor any
stored bookmark. The response `location`/`rotation` are the *input echoed
back* (the zero defaults), so a caller cannot tell the call captured nothing.
This is the canonical "silent success-with-no-effect" tool bug, and it also
**misreports** the saved transform (a result field that is not what the
description promises was saved). A later "jump back to this framing" step
cannot work: the framing was never persisted.

Note the contrast with the sibling `editor.create_bookmark`, which takes only
`index`, returns a plain `{"success":true,"index":N,"message":"Bookmark N
created"}` (no echoed transform), and is the one whose own wiki/description tell
callers to prefer it. `misc.create_bookmark`'s extra `location`/`rotation`
params and "current viewport camera transform" promise are the misleading part.

**Workaround:** use `editor.create_bookmark {index}` for an editor-context
bookmark (it does not claim to echo a transform). For an explicit transform,
note `misc.create_bookmark` does not persist one either.

**Fix:** either (a) implement the advertised behavior — when
`location`/`rotation` are omitted, read the active level-editor viewport
camera transform; store an actual bookmark (`IBookmarkTypeTools` /
`UBookmark2D`/`UBookmark`); and return a true read-back of what was stored;
or (b) if storing a real bookmark is out of scope here, stop claiming it —
drop the "Save the current viewport camera transform" language and the
`location`/`rotation` echo, and either return an explicit error or a
clearly-labeled "requested (not applied)" result, deferring to
`editor.create_bookmark`. Whichever path, the description/wiki must match the
implementation (cf. the sibling `B-set-viewport-resolution-noop`, IN-REVIEW,
which took option (b) for the analogous misc-stub no-op).

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against
  `mcp__editor-automation__call`. First read the live viewport:
  `system.inspect.get_viewport_info {}` →
  `cameraLocation:{x:236.31,y:-111.48,z:184.36}`,
  `cameraRotation:{pitch:0.4,yaw:-59.2,roll:0}`. Then the exact failing call
  `misc.create_bookmark {index:0, name:"HeroShotFraming"}` (no location/rotation)
  → success `{"index":0,"name":"HeroShotFraming","location":{"x":0,"y":0,"z":0},
  "rotation":{"pitch":0,"yaw":0,"roll":0}}` — reports origin, NOT the live
  camera the description promises to save. Source confirms the handler never
  reads the viewport and never stores a bookmark: it reads `location`/`rotation`
  from input (defaulting to zero), `UE_LOG`s, and re-serializes the input as
  success (`MiscHandler.cpp:320-358`); a body grep finds no
  viewport-client read and no `IBookmarkTypeTools`/`UBookmark` write. So the
  success + the echoed transform are unbacked: the call is a silent no-op. (A
  jump round-trip was inconclusive as an oracle here because
  `editor.jump_to_bookmark` also did not move the viewport in this session even
  for an `editor.create_bookmark`-saved slot — out of scope for this ticket; the
  source read is the authoritative confirmation that `misc.create_bookmark`
  stores nothing.) Surfaced from a slow-motion-hero-shot task (REALISM mode,
  namespace `misc`); the only blocking finding once `misc.set_viewport_resolution`
  is excluded (that method's NOT_IMPLEMENTED is already covered by
  `B-set-viewport-resolution-noop` + `E-fixed-size-capture-discovery`).
- `#2-fail-loud-not-implemented` `IN-REVIEW` developer — Applied Fix option (b),
  the accepted silent-success-class resolution (cf. `B-set-viewport-resolution-noop`
  IN-REVIEW, `B-material-stub-handlers-silent-success` DONE). Option (a)
  (collapse to the sibling via `GEditor->Exec("SetBookmark N")`) was rejected: the
  engine bookmark command captures the *live viewport camera*, so it cannot honor
  this variant's distinguishing explicit `location`/`rotation` params — wiring it
  would silently ignore those params and re-introduce a different misreport. In
  `Handlers/Utility/MiscHandler.cpp` the silent-no-op body (read location/rotation
  defaulting to zero, one `UE_LOG`, then the input-echoed `SendSuccess`) is replaced
  with `SendError("NOT_IMPLEMENTED", …)` whose message explains the method never
  read the camera/stored a bookmark and points callers at `editor.create_bookmark
  {index}` (stores the live framing) + `editor.jump_to_bookmark {index}`. The
  `REGISTER_RPC_HANDLER` description (which had falsely claimed "Save the current
  viewport camera transform to a bookmark slot") now states it returns
  NOT_IMPLEMENTED and names the alternative, so the auto-generated wiki regenerated
  at launch matches the implementation. The two namespace-level wiki overlays that
  described it as a working "near-duplicate kept for symmetry"
  (`docs/wiki-src/misc.md` See also, `docs/wiki-src/editor.md` cross-cluster
  overlap) are updated to say it is deprecated / returns NOT_IMPLEMENTED and to use
  `editor.create_bookmark`. Regression test: in
  `Private/Tests/Utility/TestUtilityHandlers.cpp` the old no-crash
  `FMiscCreateBookmarkValidParamsTest` is replaced by
  `FMiscCreateBookmarkReturnsNotImplementedTest`, which invokes the real handler the
  documented way (`{index:0, name:"HeroShotFraming"}`, no location/rotation) via
  `InvokeHandlerWithCapture` and asserts `bSuccess == false` and `ErrorCode ==
  "NOT_IMPLEMENTED"` — it would fail if the input-echoing silent-success stub were
  restored. Not compiled/run here (later phase verifies).
