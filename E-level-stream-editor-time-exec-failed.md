---
id: E-level-stream-editor-time-exec-failed
title: "level.stream fails [EXEC_FAILED] Command not executed at editor authoring time (it wraps the runtime-only StreamLevel console command); no hint that level.set_visibility is the editor-time path"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [level, stream, set_visibility, exec-failed, runtime-only, console-command, docs]
---

# `level.stream` fails cryptically at editor time and never points at the working verb

`level.stream` is a thin wrapper that builds a `StreamLevel <name> Load|Unload
Show|Hide` console command and cross-dispatches it through
`system.console_command` (`LevelHandler.cpp:470-475`):

```cpp
const FString Cmd = FString::Printf(TEXT("StreamLevel %s %s %s"), *LevelName,
    bLoad ? TEXT("Load") : TEXT("Unload"),
    bVis ? TEXT("Show") : TEXT("Hide"));
// ... -> CrossDispatchLevel(Ctx, TEXT("system.console_command"), P);
```

`StreamLevel` is a **runtime / PIE-only** console command — it is consumed by
the *game* world's streaming subsystem, not the editor world. At editor
authoring time `GEditor->Exec` finds no handler that consumes it and returns
`false`, so `SystemControlHandler.cpp:604` emits the bare
`[EXEC_FAILED] Command not executed`. The error names neither `level.stream`,
nor the underlying `StreamLevel` command, nor the fact that the command is
runtime-only — so an agent cannot tell whether it passed a bad `levelName`, hit
a transient failure, or asked for something structurally impossible in the
editor.

## Why it's ergonomic friction (not just a bug)

Unlike `B-set-view-mode-exec-failed` (DONE) — where the `viewmode` exec *could*
be routed to the active level-viewport client and made to succeed — `StreamLevel`
has **no editor-time consumer to route to**. The toggle the caller actually
wants at editor time is already served by sibling verbs:

- the **visible** flag → `level.set_visibility` (worked first try in the audited
  task), and
- the **loaded** flag → the load/unload streaming-state path.

So `level.stream` is genuinely the wrong tool for editor-time work, yet its
registration description (`LevelHandler.cpp:441`) sells it without that caveat:

> "Toggle a streaming sublevel's loaded and visible state independently. Useful
> for runtime level streaming without removing the level from the world."

"Useful for runtime level streaming" is the only hint, and it reads as a
feature, not a restriction. An agent following the obvious "toggle a sublevel's
loaded/visible state" intent reaches for `level.stream`, eats two cryptic
`[EXEC_FAILED]` failures, and only then discovers `level.set_visibility` is the
verb that works.

## What it should do

Make the editor-time dead-end self-correcting:

1. **Fail loud with a redirect.** When `level.stream` runs outside PIE/game
   (detect `GEditor->PlayWorld == nullptr` / not in a play session), short-circuit
   before dispatching `StreamLevel` and return a descriptive error
   (e.g. `RUNTIME_ONLY_COMMAND`) stating that `StreamLevel` only works at
   runtime/PIE and naming the editor-time path: `level.set_visibility` for the
   visible flag (and the load/unload verb for the loaded flag). This converts
   the cryptic `[EXEC_FAILED] Command not executed` into an actionable message.
2. **Docs:** the `level.md` wiki-src overlay already lists `stream` under the
   "legacy level-streaming model" gotcha but does NOT warn that `level.stream`
   is a runtime-only `StreamLevel` wrapper that fails `[EXEC_FAILED]` in the
   editor, nor that `level.set_visibility` is the editor-time substitute. Add
   that note to **`Docs/wiki-src/level.md`** (the wiki edit is a downstream
   process, not this ticket).

## Evidence (struggle audit, focus `level.remove_from_world`)

Streaming-sublevel attach/detach round-trip task. After attaching
`StreamingTest_Annex`, step 5 ("make it loaded-but-hidden, then loaded-and-visible")
hit the dead-end:

- `level.stream` (full path) → `[EXEC_FAILED] Command not executed`.
- `level.stream` (retry, short name `StreamingTest_Annex`) →
  `[EXEC_FAILED] Command not executed`.
- `level.set_visibility {visible:false}` → succeeded (loaded-but-hidden).
- `level.set_visibility {visible:true}` → succeeded (loaded-and-visible).

Friction note verbatim: "level.stream failed twice (EXEC_FAILED) on an
editor-time sublevel since it just wraps the runtime-only StreamLevel console
command; level.set_visibility worked." That is 2 wasted is_error calls + a
manual pivot to the correct verb for an intent the API names `stream`.

Distinct from the judge-filed `B-level-create-makes-wp-map` (the WP-map /
no-`.umap` tool bug earlier in the same task) and from `B-set-view-mode-exec-failed`
(DONE — a viewport exec that *could* be routed to succeed; `level.stream` cannot,
because `StreamLevel` has no editor-time consumer). Dedup: ripgrep across OPEN +
closed found no existing ticket on `level.stream` / `StreamLevel` editor-time
failure (only passing references in unrelated files).

## History
- `#2-stream-runtime-only-guard` `IN-REVIEW` developer — Implemented the fail-loud-with-redirect fix. In `level.stream` (`Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Level/LevelHandler.cpp`), inserted a PIE-detection guard right after the empty-name validation and before the `StreamLevel` Printf/cross-dispatch: when `GEditor->PlayWorld == nullptr` (not in a play session, the codebase's standard PIE check used at EditorCommandHandler.cpp:231 etc.), short-circuit with `Ctx.SendError(TEXT("RUNTIME_ONLY_COMMAND"), …)` whose message states StreamLevel is runtime/PIE-only and names the editor-time verbs — `level.set_visibility` for the visible flag, and `level.add_to_world`/`level.remove_from_world`/`level.structure.configure_level_streaming` for the loaded flag (honoring the adversarial lens note that there is no single `level.set_loaded` verb; the loaded path is split across those three). Also rewrote the misleading registration description (LevelHandler.cpp:444) to lead with the runtime/PIE-only caveat and the editor-time substitutes instead of the bare "Useful for runtime level streaming" hint. Added the wiki note to `Docs/wiki-src/level.md` gotchas as the ticket requested. Regression test: `EditorAutomationRpcGateway.level.stream.EditorTimeRuntimeOnlyRedirect` in `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Tests/World/TestLevelHandlers.cpp` — invokes the production `level.stream` handler via `InvokeHandlerWithCapture` outside PIE and asserts the response is an error with `ErrorCode == "RUNTIME_ONLY_COMMAND"` (not `EXEC_FAILED`) and a message naming `level.set_visibility`; would fail if the guard were reverted (code would fall through to the cross-dispatch and surface the bare `EXEC_FAILED`). All three validity lenses voted valid; cited line numbers drifted cosmetically (470-475→473-478, 441→444, 604→600) but every mechanism verified exact against current source. Not compiled/tested here (later phase).
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `level.remove_from_world` streaming-sublevel round-trip. `level.stream` failed twice with `[EXEC_FAILED] Command not executed` (full path, then short name) at editor authoring time; `level.set_visibility` then worked first try for both the hidden and visible toggles. Root cause: `level.stream` (`LevelHandler.cpp:470-475`) builds a `StreamLevel <name> Load Show` console command and cross-dispatches it via `system.console_command`; `StreamLevel` is a runtime/PIE-only command with no editor-time consumer, so `GEditor->Exec` returns false and `SystemControlHandler.cpp:604` emits the bare `EXEC_FAILED`. The registration description (`LevelHandler.cpp:441`) only hints "Useful for runtime level streaming" with no editor-time caveat, and the `Docs/wiki-src/level.md` gotcha lists `stream` as legacy-streaming without warning it fails in the editor or naming `level.set_visibility` as the substitute. Proposed: short-circuit with a `RUNTIME_ONLY_COMMAND`-style error that redirects to `level.set_visibility` when not in PIE, plus a wiki note. Evidence: 2 `level.stream` is_error calls then 2 working `level.set_visibility` calls in the call log. Dedup: no existing `level.stream`/`StreamLevel`-editor-time ticket (ripgrep OPEN+closed clean); distinct from `B-level-create-makes-wp-map` (judge-filed WP bug, same task) and `B-set-view-mode-exec-failed` (DONE, routable viewport exec).
</content>
</invoke>
