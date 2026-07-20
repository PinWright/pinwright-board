---
id: E-compile-reinstances-live-instances-no-guard
title: "blueprint.set_default / blueprint.compile reinstance live PIE instances with no warning"
status: OPEN
severity: Medium
category: ergonomic
tags: [blueprint, set-default, compile, reinstance, pie, live-instances, safety]
encounters: 1
lastSeen: 2026-07-20T00:00:00Z
---

# `blueprint.set_default` / `blueprint.compile` reinstance live PIE instances with no warning

`blueprint.set_default` finalizes by calling `FKismetEditorUtilities::CompileBlueprint`
(`Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp:485`, added by
sibling `B-blueprint-set-default-not-persisted` so CDO writes persist), and `blueprint.compile`
calls it directly. A full compile flushes the reinstancing queue
(`FBlueprintCompilationManagerImpl::FlushReinstancingQueueImpl` → `ReplaceInstancesOfClass_Inner`
→ `UWorld::EditorDestroyActor`), which **tears down every live instance of the class — including
actors in a running PIE session** — then re-creates them. Neither handler detects that PIE is
active / live instances exist, and neither warns that the call will destroy and re-spawn them.
A blind agent calling "set a default value" has no signal it just triggered a heavyweight
reinstancing pass over live PIE state.

Reinstancing-on-compile is normal, designed UE behavior and is harmless when actor teardown is
well-behaved; a human compiling a BP during PIE from the editor UI gets the identical reinstance.
It only faults when a game's teardown path is fragile. Session evidence: `set_default` of a
material default on `/App/HELIOS/Drones/Atlas/B_PioneerSumo` during a live PIE with drone
instances → compile → reinstance → `EditorDestroyActor` → (game) `ADrone::RefreshControllerChanged`
→ `UFPVCameraComponent::IsLocallyControlled` null-deref → editor ACCESS_VIOLATION.
**Scope: the null-deref was the crash's proximate cause and is fixed game-side — this ticket is
only the MCP-side gap: no warning/opt-out for reinstance-on-compile during live PIE.** Board
precedent for surfacing dangerous editor state: `B-editor-save-all-pie-diagnostic` (surface
`pieActive`) and the material open-editor guard family (`B-material-graph-edit-clobbered-by-open-editor`).

**Workaround:** Set CDO defaults / compile before starting PIE, or stop PIE first
(`ui.stop_play`); don't run `set_default` / `compile` on a class that has live PIE instances.
**Fix:** Detect a running PIE / live instances of the target class (`GEditor->PlayWorld` +
instance scan) and either return a `pieActive` / `reinstancedLiveInstances` warning field on the
success response, or gate the compile behind an explicit opt-in arg — default warn-not-block,
since reinstancing is legitimate hot-reload; a hard block would break valid compile-during-PIE.

## History
- `#1-initial-repro` `OPEN` reporter — Verified in source: `blueprint.set_default`'s finalize block calls `FKismetEditorUtilities::CompileBlueprint(Blueprint)` at `BlueprintPropertyHandler.cpp:485` with no PIE / live-instance guard or warning; `blueprint.compile` does the same. Session repro: `blueprint.set_default` (material default) on `/App/HELIOS/Drones/Atlas/B_PioneerSumo` in an editor running PIE with live drone instances triggered the compile → reinstancing queue flush → `ReplaceInstancesOfClass_Inner` → `UWorld::EditorDestroyActor` destroyed the live drones mid-call → game-code null-deref (`ADrone::RefreshControllerChanged` → `UFPVCameraComponent::IsLocallyControlled`) → editor ACCESS_VIOLATION. The game null-deref is fixed separately; filed here for the un-warned reinstance-on-compile side effect during live PIE. Checked the family — `B-bp-saved-state-corruption-mcp-edits`, `B-material-graph-edit-clobbered-by-open-editor`, `B-material-graph-mutators-bypass-editor-open-guard`, `B-bpir-macro-recompile-orphans-caller-instances`, `B-compile-save-after-compile-timeout`, and sibling `B-blueprint-set-default-not-persisted` — none cover compile-during-live-PIE reinstancing of live actors; filed NEW.
