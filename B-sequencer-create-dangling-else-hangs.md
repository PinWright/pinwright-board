---
id: B-sequencer-create-dangling-else-hangs
title: "SequenceHandler.cpp file-wide dangling-else (70 braceless guards) → every Sequencer method hangs/times out on its valid path, responds nothing"
status: IN-REVIEW
severity: High
category: bug
tags: [sequencer, create, set_display_rate, hang, timeout, no-response, dangling-else, dead-code, missing-braces, file-wide]
---

# SequenceHandler.cpp file-wide dangling-else → every Sequencer method hangs/times out on its valid path

`sequencer.create` with the documented-valid params (`name` required, `path`
optional — exactly as the wiki page lists) **never responds**: the MCP call
hangs and the client surfaces "The operation timed out", and **no asset is
created**. The editor stays fully responsive to every other call throughout
(this is a handler-level no-response, not editor-down). This makes the
documented level-sequence authoring entrypoint 100% non-functional.

**This is not an isolated bug.** Source verification (a full scan of
`SequenceHandler.cpp`) shows the identical dangling-else defect at **70
braceless guard sites**, spanning effectively every handler in the file
(`sequencer.create`, `set_display_rate`, `set_properties`, `add_actor`,
`add_actors`, `add_spawnable_from_class`, `set_playback_speed`, `duplicate`,
`rename`, `delete`, `add_keyframe`, `set_track_muted`/`solo`/`locked`,
`add_track`, `add_section`, and more). For every guard that is a handler's
**first** statement, the unconditional `return true;` fires on the valid path
too, so that whole method hangs identically to `sequencer.create`. The true
blast radius is the **entire Sequencer namespace**, not one method — High
severity is justified and arguably understated.

## Root cause

A braceless `if` (dangling-else / missing-braces bug) at
`Source/EditorAutomationRpcGateway/Private/Handlers/Sequencer/SequenceHandler.cpp:208-210`:

```cpp
FString Name = Ctx.GetString("name");
FString Path = Ctx.GetString("path");
if (Name.IsEmpty())
    Ctx.SendError("INVALID_ARGUMENT", "sequence_create requires name");
    return true;          // <-- UNCONDITIONAL: runs on every path
```

Because the `if` has no braces, only the `SendError` line is guarded; the
`return true;` is unconditional. On the normal valid path (`Name` non-empty)
the handler skips `SendError` and immediately `return true;` **without ever
calling `Ctx.SendSuccess` / `Ctx.SendError`**. Everything from line 212
onward — the `LevelSequenceFactoryNew` lookup, `CreateAsset`, save, and the
success response (lines 212-261) — is **dead, unreachable code**.

A handler that returns `true` without sending a response leaves the transport
waiting for a completion that never arrives, so the caller observes a generic
timeout while the editor itself is fine. This is exactly the
`NO_HANDLER_RESPONSE` scenario that `F-no-response-handler-guardrail` flagged
as "regression-only / theoretical" — here it is a real, live, shipping handler
hitting it. (The dispatcher guardrail in that ticket would have turned this
silent hang into a diagnosable `NO_HANDLER_RESPONSE`, but the guard is gated to
the non-production capture path, so it does not fire for live traffic — the
hang is what callers actually see.)

The same braceless-`if` + unconditional-`return true;` pattern repeats in this
file at `sequencer.set_display_rate` (`SequenceHandler.cpp:276-278` and
`281-283`) and at **68 other sites** (confirmed by file scan, 70 total). Because
line 278's `return true;` is the first guard's unconditional return,
`set_display_rate` is **definitely broken** (not "likely") — every valid
(non-empty-path) call returns without a response and hangs, identical to
`sequencer.create`. The file-wide sweep is **mandatory, not advisory**: this is
a single systemic dangling-else defect, not an isolated pair.

## Repro (live, this session, via mcp__editor-automation__call)

- `sequencer.create` `{name:"SEQ_ReplayProbe", path:"/Game/FuzzReplay"}` (both
  documented params; target asset confirmed non-existent first via
  `asset.exists` → `exists:false`) → **"The operation timed out"** (hang).
- Immediately after, `system.inspect.get_world_settings {}` →
  `{worldName:"MRQTestMap", levelName:"PersistentLevel", success:true}`
  (editor fully alive — not editor-down).
- `asset.exists {assetPath:"/Game/FuzzReplay/SEQ_ReplayProbe"}` →
  `exists:false` (nothing was created despite the timeout).
- The originating attempt reproduced the same hang 4× with
  `{name:"SEQ_MRQTest", path:"/Game/Cinematics"}` / name-only and had to
  author the sequence via `python.execute` as a workaround.

## Impact

The only documented RPC to create a level sequence is unusable — every call
hangs ~120 s (the MCP client timeout) and writes nothing, blocking any
Sequencer/MRQ authoring workflow at step one. Agents must fall back to
`python.execute`.

**Workaround:** Author the level sequence via `python.execute`
(`LevelSequenceFactoryNew` + `AssetTools.create_asset`) instead of
`sequencer.create`.

**Fix:** Wrap the body of every braceless guard of the form `if (Cond)
Ctx.Send...(...); return true;` in braces so `return true;` runs only after the
error is sent. This must be applied to **all 70 sites** in
`SequenceHandler.cpp` (a single file-wide sweep), not just `sequencer.create`
and `set_display_rate` — they are one and the same systemic defect. Consider
enabling `-Wdangling-else` / requiring braces on single-statement guards to
catch this class statically.

## History
- `#1-initial-repro` `OPEN` reporter — `sequencer.create {name:"SEQ_ReplayProbe", path:"/Game/FuzzReplay"}` (documented-valid params, target confirmed absent) timed out / hung with no asset created; editor stayed live (`system.inspect.get_world_settings` succeeded immediately after, `asset.exists`→false). Root cause is a braceless `if (Name.IsEmpty())` at `SequenceHandler.cpp:208-210` whose unconditional `return true;` makes the handler return without calling SendSuccess/SendError on the valid path, leaving lines 212-261 (factory/CreateAsset/save/success) as dead code → transport waits forever → client sees timeout. Originating attempt hit the same hang 4× and worked around it via python.execute. Same braceless pattern also present at sequencer.set_display_rate (276-278, 281-283).
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded (title/body/Fix/tags) from "single-handler bug + 2 likely siblings" to the confirmed file-wide scope: a full scan of `SequenceHandler.cpp` found **70** braceless `if (Cond) Ctx.Send...(...); return true;` guard sites (the `return true;` unconditional), so `set_display_rate` and ~every other Sequencer method also hang on their valid path — not just `sequencer.create`. **Fix:** wrapped the body of all 70 guards in braces so `return true;` runs only after the error is sent, restoring each handler's success path. Single file changed: `Source/EditorAutomationRpcGateway/Private/Handlers/Sequencer/SequenceHandler.cpp` (brace-only; no logic moved). Verified post-edit that 0 dangling sites remain and overall brace balance is unchanged. **Test:** added `Private/Tests/Sequencer/TestSequenceHandlerGuardResponse.cpp` — 3 automation tests that drive the real registered `sequencer.create` (valid path + empty-name path) and `sequencer.set_display_rate` (valid path) handlers through `InvokeHandlerWithCapture` (production code) and assert `Capture.bWasCalled` (the handler actually sent a response). Counterfactual: revert the braces and the valid path returns true without sending → `bWasCalled` stays false → these tests fail. Not yet compiled/run (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 1 citation sits in history rows and is left verbatim per the append-only rule. **Deliberately not repointed — repointing would accuse innocent code.** `:208-210` at HEAD is `EnsureSequenceEntry`'s benign one-line guard `if (SeqPath.IsEmpty()) return nullptr;`, not the reported braceless-`if` bug. The `sequence.create` name guard the ticket is about is `Handlers/Sequencer/SequenceHandler.cpp:457-461` and is **correctly braced**; the file's only `"sequence_create requires name"` is `:459`. So the defect is fixed and the old line now points at unrelated compiling code — the worst failure mode for a citation. Ticket is still IN-REVIEW. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
