---
id: F-editor-status
title: "Add `editor.status` read-only probe for PIE / editor state"
status: DONE
severity: Low
category: feature
tags: [editor, pie, ergonomic]
---

# Add `editor.status` read-only probe for PIE / editor state

No MCP method currently reports whether PIE is running or returns
general editor state. `editor.status` does not exist
(`call("editor.status")` → `UNKNOWN_ACTION: Unknown action: editor.status`).

The only ways to discover PIE state are:

- Call `editor.play` with empty args — returns `{success: true,
  alreadyPlaying: true}` when PIE is live, **but starts PIE as a side
  effect when it is not**. Same chicken-and-egg with `editor.stop`. Neither
  is safe as a pre-flight probe.
- `python.execute` boilerplate around
  `unreal.UnrealEditorSubsystem.get_game_world()` — works but is a
  full Python round-trip for a one-line state query.
- After `B-inspect-misses-pie-world` (DONE) callers can issue
  `system.inspect.list_objects { world: "pie" }` and read the echoed
  `worldPath`; a `UEDPIE_` prefix implies PIE is active. This works as
  an indirect probe but requires running an actor enumeration just to
  read one bit.

Proposed read-only handler:

```json
{
  "inPie": true,
  "pieIsPaused": false,
  "pieWorldPath": "/Game/.../UEDPIE_0_L_Foo.L_Foo",
  "editorWorldPath": "/Game/.../L_Foo.L_Foo",
  "playerControllerPath": "/Game/.../UEDPIE_0_L_Foo.L_Foo:PersistentLevel.B_DronePlayerController_C_0",
  "elapsedPieSeconds": 12.3
}
```

No mutation, no side effect. Implementation walks
`GEditor->GetWorldContexts()` for the `EWorldType::PIE` entry (same
helper that `ResolveQueryWorld` uses post-`B-inspect-misses-pie-world`)
and reads `GEditor->PlayWorld`, the editor world, the
`UWorld::TimeSeconds`, and the first PlayerController.

**Relation to `B-inspect-misses-pie-world`:** that fix partially solves
the same need (callers can infer PIE via `worldPath` echo on inspect
calls). `editor.status` is the dedicated zero-side-effect probe and
removes the need for inspect-as-probe and `python.execute` boilerplate.

**Workaround:** Issue `system.inspect.list_objects { world: "pie" }`
and check whether `worldPath` contains `UEDPIE_`, or `python.execute`
with `UnrealEditorSubsystem.get_game_world()`.

**Fix:** New handler `editor.status` in `Handlers/Editor/` returning
the JSON shape above. Reuse `McpActorUtils::ResolveQueryWorld` (or a
sibling helper that returns both editor + PIE worlds without
preferring one). No params.

## History
- `#1-feature-request` `OPEN` reporter — No read-only probe for PIE/editor state; `editor.play`/`editor.stop` are destructive when used as probes; `B-inspect-misses-pie-world`'s `worldPath` echo gives an indirect signal but requires running an actor enumeration. Severity Low — workarounds exist (inspect echo, `python.execute`).
- `#2-handler-added` `IN-REVIEW` developer — Added `editor.status` handler in `PIEHandler.cpp`: returns `inPie`, `pieIsPaused`, `pieWorldPath`, `editorWorldPath`, `playerControllerPath`, `elapsedPieSeconds`. Walks `GEditor->GetWorldContexts()` for the PIE entry (mirroring `McpActorUtils::ResolveQueryWorld`). No mutation. Regression: `EditorAutomationRpcGateway.editor.status.Registered` + `.ValidParamsNoCrash` (TestEditorHandlers.cpp).
- `#3-verify-status-probe` `DONE` tester — Verified: `call("editor.status")` documents no parameters and `call("editor.status", {})` returned `inPie: false`, `pieIsPaused: false`, empty PIE/player controller paths, editor world `/Game/System/FrontEnd/Maps/L_Core.L_Core`, and `elapsedPieSeconds: 0`.
