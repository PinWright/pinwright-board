---
id: B-inspect-settings-stats-stub-silent-success
title: "system.inspect.get_editor_settings / get_project_settings / get_performance_stats / get_memory_stats are no-op stubs that return success with a 'retrieved' message and no data"
status: IN-REVIEW
severity: Medium
category: bug
tags: [system-inspect, stub, silent-success, no-effect, settings, performance, memory]
---

# `system.inspect` settings/stats readers return `success:true` with no data

Four registered `system.inspect` readers report success while returning **zero**
of the data their name and message promise. They never enumerate anything:

- **`system.inspect.get_editor_settings`** → `{"action":"get_editor_settings","message":"Editor settings retrieved","success":true}` — no settings/preferences fields at all. The message asserts settings were "retrieved" when none are returned.
- **`system.inspect.get_project_settings`** → `{"action":"get_project_settings","message":"Project settings retrieved","success":true}` — same shape, no project settings returned.
- **`system.inspect.get_performance_stats`** → `{"success":true,"message":"Performance stats placeholder - implement with actual metrics"}` — no metrics.
- **`system.inspect.get_memory_stats`** → `{"success":true,"message":"Memory stats placeholder - implement with actual metrics"}` — no metrics.

This is the silent-success-with-no-effect / stub-returns-success bug class already
accepted on this board: `B-material-stub-handlers-silent-success` (DONE),
`B-export-snapshot-empty-stub` (IN-REVIEW), `B-input-trigger-modifier-stub-silent-success`.
As in those tickets, the wiki *does* disclose these as stubs (`system.inspect.md`:
"`get_project_settings`, `get_editor_settings`, `get_performance_stats`,
`get_memory_stats`, and `get_scene_stats` currently return stub responses"; the
`get_editor_settings` page: "Return a placeholder summary that editor settings
were retrieved. Currently does not enumerate editor preferences"). The accepted
board position is that wiki disclosure does **not** excuse a `success:true`
response that the wire contract makes indistinguishable from a real read — a
schema-driven caller (or an agent doing a read-only "environment audit": capture
editor preferences + project settings + perf/memory) gets a green checkmark and an
empty payload, with `get_editor_settings`/`get_project_settings` actively claiming
the data was "retrieved".

`get_editor_settings` is the worst offender: its message ("Editor settings
retrieved") states a falsehood, whereas the perf/memory pair at least self-label
"placeholder - implement with actual metrics".

Note: the same wiki line lists `get_scene_stats` as a stub, but on replay
`system.inspect.get_scene_stats` returns real data (`{"actorCount":227,"success":true}`),
so it appears implemented — exclude it from this bug (the wiki note is stale for
that one method; a doc nit, not part of this fix).

**Workaround:** for real inspection data use the implemented readers
(`get_world_settings`, `get_scene_stats`, `list_subsystems`, `get_viewport_info`,
`get_selected_actors`, `inspect_object`); there is currently no RPC that returns
editor preferences, project settings, or perf/memory metrics.

**Fix (matching the accepted stub precedent — minimum safe change):** replace the
`success:true` no-op in each of the four handlers with
`SendError("NOT_IMPLEMENTED", "...")` so the contract is honest and the no-op is
not disguised as a successful read. (Implementing real settings/metrics
enumeration would close the capability gap, but the one-line honest-failure swap
is the precedent-matching minimum, same as B-material-stub-handlers-silent-success
and B-export-snapshot-empty-stub.) Also drop `get_scene_stats` from the stub list
in `system.inspect.md` since it is implemented.

## Verbatim repro (live, replay-confirmed against `mcp__editor-automation__call`)

Fresh editor open, world `ExampleProjectWelcome` / `PersistentLevel`.

1. `system.inspect.get_editor_settings` `{}`
   → `{"action":"get_editor_settings","message":"Editor settings retrieved","success":true}`
2. `system.inspect.get_project_settings` `{}`
   → `{"action":"get_project_settings","message":"Project settings retrieved","success":true}`
3. `system.inspect.get_performance_stats` `{}`
   → `{"success":true,"message":"Performance stats placeholder - implement with actual metrics"}`
4. `system.inspect.get_memory_stats` `{}`
   → `{"success":true,"message":"Memory stats placeholder - implement with actual metrics"}`

For contrast, the implemented siblings on the same world return real payloads:
`get_world_settings` → `{"worldName":"ExampleProjectWelcome","levelName":"PersistentLevel","success":true}`;
`get_scene_stats` → `{"actorCount":227,"success":true}`.

## History
- `#2-honest-failure-swap` `IN-REVIEW` developer — Applied the precedent-matching minimum fix (same as B-material-stub-handlers-silent-success DONE and B-export-snapshot-empty-stub IN-REVIEW): swapped the four success-returning no-ops to `Ctx.SendError("NOT_IMPLEMENTED", ...)` so the dishonest `success:true` contract is gone. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp` — `system.inspect.get_project_settings`, `get_editor_settings`, `get_performance_stats`, `get_memory_stats` now SendError NOT_IMPLEMENTED (with hints pointing project-settings callers at `rendering.get_project_settings` and perf/memory callers at the `performance.*` namespace); their REGISTER_RPC_HANDLER summaries updated from "placeholder/retrieved" to "Not implemented". `get_scene_stats` left untouched (it genuinely returns `actorCount`). Wiki: `Docs/wiki-src/system.inspect.md` stub note rewritten — the four readers now documented as NOT_IMPLEMENTED and `get_scene_stats` removed from the stub list / noted as implemented. Regression test: new `Source/EditorAutomationRpcGateway/Private/Tests/Environment/TestSystemInspectSettingsStatsNotImplemented.cpp` — four `IMPLEMENT_SIMPLE_AUTOMATION_TEST` cases drive each handler via the production registration (`InvokeHandlerWithCapture` → `Reg.Func`) and assert `bSuccess==false` + `ErrorCode=="NOT_IMPLEMENTED"`; each fails if a handler is reverted to SendSuccess. Existing `FSystemInspectGetProjectSettingsValidParamsTest` still passes (it only asserts the handler is found, which still returns true). Not yet compiled/tested — a later phase verifies green.
- `#1-initial-repro` `OPEN` reporter — Surfaced by a read-only "environment audit" task (seed `system.inspect.get_editor_settings`). Replay-confirmed live via `mcp__editor-automation__call`: `get_editor_settings {}` → `{"action":"get_editor_settings","message":"Editor settings retrieved","success":true}` (no settings), `get_project_settings {}` → `{"action":"get_project_settings","message":"Project settings retrieved","success":true}` (no settings), `get_performance_stats {}` → `{"success":true,"message":"Performance stats placeholder - implement with actual metrics"}`, `get_memory_stats {}` → `{"success":true,"message":"Memory stats placeholder - implement with actual metrics"}`. All four are success-returning no-ops that emit none of the data their name/message promise; `get_editor_settings`/`get_project_settings` additionally claim the data was "retrieved". Same bug class as B-material-stub-handlers-silent-success (DONE) and B-export-snapshot-empty-stub (IN-REVIEW); wiki disclosure of the stubs does not make the `success:true` wire contract honest. Precedent fix: swap `SendSuccess` → `SendError("NOT_IMPLEMENTED", ...)` in each handler. Sibling `get_scene_stats`, listed as a stub in the same wiki note, actually returns real data (`actorCount:227`) and is excluded.
