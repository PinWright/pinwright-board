---
id: E-editor-camera-readback-not-discoverable-from-focus-actor
title: "editor.* camera-moving verbs (focus_actor / set_camera / jump_to_bookmark) and editor.status give no pointer to the camera read-back, which lives in a different namespace (system.inspect.get_viewport_info) — the existing ### editor.focus_actor overlay section omits it, so callers grep the wiki to find where the camera transform lives"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [editor, focus_actor, set_camera, jump_to_bookmark, status, viewport, camera, readback, get_viewport_info, cross-namespace, discoverability, docs]
---

# `editor.focus_actor` / `editor.status` give no pointer to the camera read-back, which lives in a different namespace (`system.inspect.get_viewport_info`)

After framing an actor with `editor.focus_actor`, the natural next step in any
"frame it, then note the resulting camera location/rotation" task is to read the
viewport camera transform back. But:

- `editor.focus_actor` returns only `{"success":true}` — no resulting camera
  location/rotation, even though the whole point of "focus on it so the camera
  sits at a sensible distance" is that the camera moved to a new transform the
  caller wants recorded.
- `editor.status` (the obvious "tell me the editor's current state" verb) does
  **not** include the viewport camera transform either.

The actual camera read-back exists and works — but it lives in a **different
namespace**, `system.inspect.get_viewport_info` (which now returns
`cameraLocation` / `cameraRotation` / `fov` per the DONE ticket
`E-viewport-info-camera-transform`). Nothing on the `editor.*` side points there.
A caller who has the camera-moving verbs in hand (`editor.focus_actor`,
`editor.set_camera`) has no signpost from those methods, or from
`editor.status`, telling them the read partner is `system.inspect.get_viewport_info`.

So the capability is present; the **discovery path is missing**. The
`docs/wiki-src/editor.md` overlay *does* have a `### editor.focus_actor` section
(editor.md:36-40), but it only covers identifier resolution (label / internal
name / object path) — it says nothing about reading the resulting camera back.
The overlay's `### editor.screenshot` section *does* helpfully name where to look
for related behavior (it points to `render.capture_open_level` /
`misc.set_viewport_resolution`), so the "name where to look" precedent exists —
but nothing in the editor viewport-camera cluster
(`set_camera` / `focus_actor` / `jump_to_bookmark` / `set_view_mode`) or in the
`editor.status` description mentions the `system.inspect.get_viewport_info`
read-back at all, and `system.inspect.md` has no section pointing back the other
way either.

This is a docs/ergonomic gap, not a bug. It is distinct from:
- `E-viewport-info-camera-transform` (DONE) — that **added** the camera fields to
  `get_viewport_info` (the producer side). It did not add any pointer *from* the
  `editor.*` camera-moving methods to that read-back, so a caller starting in
  `editor.*` still can't find it.
- `E-focus-actor-rejects-internal-name-label-only` (IN-REVIEW) — that is about
  which identifier `actorName` accepts (label vs internal name); its fix already
  landed in the handler (ViewportHandler.cpp:146 resolves via
  `McpActorUtils::FindActorByName`) and authored the existing
  `### editor.focus_actor` overlay section. Unrelated to where the resulting
  camera transform is read back — that section carries no read-back pointer.
- `B-editor-bookmark-roundtrip-noop` (IN-REVIEW, judge-filed for this same task)
  — the bookmark round-trip no-op. Separate failure; this ticket is purely the
  read-back discovery friction the agent hit *before* the bookmark steps.

## What it should do

Pick the docs path (no code change required, the read-back already exists). Note
the `### editor.focus_actor` section **already exists** (editor.md:36-40, authored
by `E-focus-actor-rejects-internal-name-label-only`); the fix **augments** it
rather than creating a duplicate H3:

1. **Augment** the existing `### editor.focus_actor` section in
   `docs/wiki-src/editor.md`, noting that the call only reports `{success:true}`
   and that **to read the resulting camera transform, call
   `system.inspect.get_viewport_info`** (returns
   `cameraLocation` / `cameraRotation` / `fov`).
2. **Add** the same read-back pointer to the symmetric camera-moving verbs that
   have no per-method section today — add a `### editor.set_camera` and a
   `### editor.jump_to_bookmark` section (both return only `{success:true}` and
   the caller wants to confirm the resulting pose numerically), each naming
   `system.inspect.get_viewport_info` as the read partner.
3. **Add** a `### editor.status` section (or one line in its context) clarifying
   it carries only PIE/world state — "the viewport camera transform is read via
   `system.inspect.get_viewport_info`, not `editor.status`."
4. Close the loop from the other side: add a `### system.inspect.get_viewport_info`
   section to `docs/wiki-src/system.inspect.md` naming it as the read partner for
   the `editor.*` camera-moving verbs (this is what an agent landed on
   `get_viewport_info` should see if it arrives there first).

A cheaper-still alternative (code, out of scope for this proposal) would be to
echo the resulting `cameraLocation`/`cameraRotation` in the `editor.focus_actor`
success payload, removing the cross-namespace hop entirely — but the docs pointer
is the minimal fix.

## Evidence (this task — viewport-bookmark "camera tour", seed `editor.create_bookmark`)

33-call workflow. Friction note, verbatim: *"Discoverability friction: neither
editor.focus_actor (returns only {success:true}) nor editor.status returns the
viewport camera transform that the task/success-check implied they would; I had
to grep the wiki to find system.inspect.get_viewport_info as the actual camera
read-back."*

The call-log shows the cost: the pre-flight wiki-nav phase read docs for
`editor.focus_actor`, `editor.status`, `actor.list`, **and**
`system.inspect.get_viewport_info` (4 doc reads) before the first execute — the
last only after grepping to discover it is the camera read-back. Every
post-framing read in the run then routes through
`system.inspect.get_viewport_info` (10 such reads), confirming it was the
load-bearing read-back the `editor.*` methods never advertised.

**Workaround:** after `editor.focus_actor` / `editor.set_camera`, read the
viewport camera with `system.inspect.get_viewport_info` (`cameraLocation`,
`cameraRotation`, `fov`); `editor.status` does not carry it.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a viewport-bookmark "camera tour" task (seed `editor.create_bookmark`; 33 calls; OUTCOME tool_bug, judge filed `B-editor-bookmark-roundtrip-noop` for the bookmark no-op). This is the distinct PROCESS angle: the camera read-back the task needed after framing is not discoverable from the `editor.*` side — `editor.focus_actor` returns only `{success:true}`, `editor.status` omits the camera, and the working read-back lives in a different namespace (`system.inspect.get_viewport_info`). The agent had to grep the wiki to find it (4 wiki-nav doc reads incl. `get_viewport_info` before first execute; 10 subsequent reads all routed there). Dedup (ripgrep over OPEN+closed; qmd unavailable): only `E-focus-actor-rejects-internal-name-label-only` (label vs internal-name acceptance — unrelated) and `B-set-camera-no-viewport-redraw` mention `focus_actor`; `E-viewport-info-camera-transform` (DONE) added the read-back fields but no pointer from `editor.*`; `B-editor-bookmark-roundtrip-noop` is the separate bookmark bug. Proposed: add a `### editor.focus_actor` section to `docs/wiki-src/editor.md` (no per-method section exists today) naming `system.inspect.get_viewport_info` as the camera read-back, and add the same pointer to the `editor.set_camera` / `editor.status` context.
- `#2-additional-realism-undiscovered` `OPEN` reporter — Additional evidence (REALISM-mode visual-QA task: open level → Lit/game-view → set_camera establishing + create_bookmark 0 → set_camera detail + create_bookmark 1 → jump_to_bookmark 0 → show_stats/screenshot/hide_stats → PIE play/stop; 30 calls, all `ok`, OUTCOME clean/agent_fail). The discovery path is so weak that this run never found the read-back at all: the agent concluded verbatim *"no editor.get_camera / viewport-camera read-back exists anywhere in the API, so jump_to_bookmark(0) restoring the saved transform can only be confirmed by the call's success message, not by reading back numeric location/rotation."* — i.e. it gave up assuming the capability was absent rather than (as in `#1`) grepping to find it. Strengthens the ticket two ways: (a) `editor.jump_to_bookmark` is another `editor.*` camera-moving verb (alongside `set_camera`/`focus_actor`) whose result a caller wants to confirm numerically and gets no pointer from; (b) without the signpost the read-back is not merely costly to find but effectively invisible. Replay-confirmed the capability is present and correct: live `system.inspect.get_viewport_info {}` → `{width:957,height:525,cameraLocation:{x:509.0,y:80.0,z:172.2},cameraRotation:{pitch:0,yaw:-90.0,roll:0},fov:90,success:true}`. No code bug (every call clean); pure discoverability/docs friction — same fix as `#1` (add the `system.inspect.get_viewport_info` pointer to the `editor.set_camera`/`jump_to_bookmark`/`status` context).
- `#3-reword-and-fix` `IN-REVIEW` developer — Reworded then implemented. Validity check (3 lenses: correctness=reword, adversarial=reword, board-historian=valid) confirmed the discoverability gap is real and code-confirmed (`editor.focus_actor` ViewportHandler.cpp:151-153, `editor.set_camera` :189-191/:205-207, and `editor.jump_to_bookmark` EditorCommandHandler.cpp:909 all return only `{success:true}`; `editor.status` PIEHandler.cpp:274-281 carries only PIE state; the read-back `system.inspect.get_viewport_info` returns cameraLocation/cameraRotation/fov at EnvironmentHandler.cpp:1283-1287), but the ticket's load-bearing premise was STALE: a `### editor.focus_actor` overlay section ALREADY exists (editor.md:36-40, authored by `E-focus-actor-rejects-internal-name-label-only`, now IN-REVIEW not OPEN) — so the fix AUGMENTS it rather than creating a duplicate H3. Reworded title/body/**Fix:**/tags accordingly and refreshed the stale OPEN cross-refs (`E-focus-actor-…` and `B-editor-bookmark-roundtrip-noop` are both IN-REVIEW). Docs-only fix: augmented `### editor.focus_actor` and added `### editor.set_camera` / `### editor.jump_to_bookmark` / `### editor.status` sections in `docs/wiki-src/editor.md`, each naming `system.inspect.get_viewport_info` (cameraLocation/cameraRotation/fov) as the camera read partner and noting `editor.status` does not carry the camera; added the reverse pointer in a new `### system.inspect.get_viewport_info` section in `docs/wiki-src/system.inspect.md`. No handler/transport code touched. Regression test: `Source/PinWright/Private/Tests/Infra/TestEditorCameraReadbackDiscoveryDocs.cpp` (5 tests) renders each per-method page through the live `WikiHandler::RenderPage` and asserts the read-back pointer survives in each H3 — reverting the overlay edits fails them. Did not compile/run (later phase).
