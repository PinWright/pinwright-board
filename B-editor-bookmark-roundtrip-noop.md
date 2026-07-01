---
id: B-editor-bookmark-roundtrip-noop
title: "editor.create_bookmark + editor.jump_to_bookmark are silent no-ops: jump returns success but never moves the viewport camera"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, bookmark, viewport, camera, silent-success, no-op, exec-command]
---

# editor.create_bookmark + editor.jump_to_bookmark are silent no-ops: jump returns success but never moves the viewport camera

`editor.jump_to_bookmark` advertises (handler description + auto-generated wiki):
*"Move the viewport camera to a previously stored bookmark slot (0..9)."* And
`editor.create_bookmark` advertises *"Save the current viewport camera transform
into one of the 10 indexed level bookmarks (Ctrl+0..9 in the editor)."* Both
return a confident `{"success":true, "message":"..."}`, but the **round-trip is
a no-op**: after `create_bookmark` stores a framing and the camera is moved
away, `jump_to_bookmark` does **not** move the viewport camera back. No slot
ever restores its framing, so any "camera tour / hop between bookmarks" task is
impossible despite both calls reporting success.

This is the canonical "silent success-with-no-effect" tool bug. A caller cannot
tell the jump did nothing: the response is `{"success":true,"index":N,
"message":"Jumped to bookmark N"}` regardless of whether the camera moved (or
whether the slot was ever populated).

**Root cause (source).** Both handlers in
`Handlers/Editor/EditorCommandHandler.cpp` just fire an editor console command
via `GEditor->Exec` and then unconditionally `SendSuccess`, ignoring the `Exec`
return value and never reading back the viewport:
- `editor.create_bookmark` (`EditorCommandHandler.cpp:854-873`):
  `GEditor->Exec(World, *FString::Printf(TEXT("SetBookmark %d"), Index));` then
  success.
- `editor.jump_to_bookmark` (`EditorCommandHandler.cpp:880-899`):
  `GEditor->Exec(World, *FString::Printf(TEXT("JumpToBookmark %d"), Index));`
  then success.

The `SetBookmark`/`JumpToBookmark` console commands are part of the level-editor
viewport input chain, not global `UEngine::Exec` verbs; routed through
`GEditor->Exec` here they do nothing (and `Exec` returns false), but the handler
discards that and reports success anyway. (Contrast the working
`editor.set_camera`, which moves the camera by writing the viewport client
directly — verified live in this same session, see history.)

This also matters for the in-review fix of `B-misc-create-bookmark-silent-noop`:
that ticket's accepted resolution makes `misc.create_bookmark` return
NOT_IMPLEMENTED and **redirects callers to `editor.create_bookmark {index}` +
`editor.jump_to_bookmark {index}` as the working alternative** — but those are
themselves silently broken, so the redirect points at a no-op.

**Workaround:** drive the camera explicitly with `editor.set_camera {location,
rotation}` and store the per-stop transforms yourself (read each via
`system.inspect.get_viewport_info`), restoring them with `editor.set_camera`
instead of bookmarks. The bookmark slots cannot be relied on for round-trip.

**Fix:** make the bookmark commands actually take effect — either route through
the active level-editor viewport client's bookmark API
(`IBookmarkTypeTools::Get().CreateOrSetBookmark(...)` /
`JumpToBookmark(Index, ..., ViewportClient)` against
`GCurrentLevelEditingViewportClient` / the active `FLevelEditorViewportClient`),
or, if the underlying mechanism truly cannot work in this automation context,
check the `Exec` result / read back the camera and return a real error instead
of fake success. Whichever path, success must mean the camera moved (jump) and a
real bookmark was stored (create); the description/wiki must match. (cf. the
silent-success class: `B-misc-create-bookmark-silent-noop` IN-REVIEW,
`B-set-viewport-resolution-noop` IN-REVIEW.)

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live (twice) against
  `mcp__editor-automation__call` in a Content Examples map. Controlled
  round-trip: (1) `editor.set_camera {location:{777,222,333},
  rotation:{pitch:-20,yaw:120,roll:0}}` → `{"success":true}`;
  `system.inspect.get_viewport_info {}` confirms
  `cameraLocation:{777,222,333}, cameraRotation:{-20,120,0}`. (2)
  `editor.create_bookmark {index:0}` → `{"success":true,"index":0,
  "message":"Bookmark 0 created"}`. (3) move away —
  `editor.set_camera {location:{5000,5000,5000}, rotation:{pitch:-90,yaw:0,
  roll:0}}` → `{"success":true}`; `get_viewport_info` confirms camera now at
  `{5000,5000,5000}`. (4) `editor.jump_to_bookmark {index:0}` →
  `{"success":true,"index":0,"message":"Jumped to bookmark 0"}`, but
  `get_viewport_info` still reports `cameraLocation:{5000,5000,5000},
  cameraRotation:{-90,0,0}` — the camera did **not** return to the bookmarked
  `{777,222,333}`. A second `jump_to_bookmark {index:0}` likewise returned
  success with the camera still at `{5000,5000,5000}`. `editor.set_camera` +
  `get_viewport_info` round-trip live and correctly in the same session, so the
  read-back path is sound; the bookmark round-trip is the broken part. Source
  confirms the cause: both handlers fire `GEditor->Exec` with
  `SetBookmark N` / `JumpToBookmark N` and unconditionally `SendSuccess`,
  ignoring the `Exec` result and never touching the viewport client
  (`EditorCommandHandler.cpp:854-899`). Distinct from
  `B-misc-create-bookmark-silent-noop` (that is the `misc.*` variant with
  echoed location/rotation; its history explicitly scoped the `editor.*` jump
  round-trip OUT). Surfaced from a viewport-bookmark "camera tour" task
  (SEED mode, seed `editor.create_bookmark`).
- `#2-fix` `IN-REVIEW` developer — Replaced the no-op `GEditor->Exec("SetBookmark
  N"/"JumpToBookmark N")` path in both handlers with the real engine bookmark API,
  routed through the active level-editor viewport client (resolved the same way the
  working `editor.set_camera` / `ForceRedrawActiveViewport` do — via
  `GEditor->GetActiveViewport()->GetClient()`, requiring `Client->GetWorld()`).
  `editor.create_bookmark` now calls
  `IBookmarkTypeTools::Get().CreateOrSetBookmark(index, client)` and verifies with
  `CheckBookmark` (returns `VIEWPORT_NOT_AVAILABLE` / `BOOKMARK_SET_FAILED` on
  failure instead of fake success). `editor.jump_to_bookmark` first guards an empty
  slot with `CheckBookmark` (honest `BOOKMARK_EMPTY` error, no fake success), then
  calls `IBookmarkTypeTools::Get().JumpToBookmark(index,
  MakeShared<FBookmarkJumpToSettings>(), client)` and invalidates+draws the
  viewport. This mirrors the engine's own bookmark recall (SLevelViewport.cpp /
  BookmarkScoped.cpp). Files:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Editor/EditorCommandHandler.cpp`
  (added includes `Bookmarks/IBookmarkTypeTools.h`, `Engine/BookmarkBase.h`,
  `Engine/BookMark.h`, `EditorViewportClient.h`; new `ResolveBookmarkViewportClient`
  helper; rewrote both handler bodies). Regression test:
  `Source/EditorAutomationRpcGateway/Private/Tests/EditorOps/TestEditorHandlers.cpp`
  — new `FEditorBookmarkRoundTripTest`
  (`EditorAutomationRpcGateway.editor.bookmark.RoundTripMovesCamera`): with a live
  level viewport it drives a full set-camera-A → create_bookmark → move-to-B →
  jump_to_bookmark round-trip and asserts the camera returned to A (fails under the
  reverted Exec no-op, which leaves it at B); headless (no viewport) it asserts
  `jump_to_bookmark` on a fresh slot returns an error, not the old fake
  `{success:true}`. Note: this also reconciles the IN-REVIEW
  `B-misc-create-bookmark-silent-noop` redirect, which now points at working verbs.
