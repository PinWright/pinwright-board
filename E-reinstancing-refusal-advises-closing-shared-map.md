---
id: E-reinstancing-refusal-advises-closing-shared-map
title: "LIVE_INSTANCES_WOULD_BE_REINSTANCED tells agents to 'close the map' (another agent's map, in a shared editor) and never names the instances, so a 0-actor refusal looks like a false positive"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, compile, reinstance, live-instances, error-message, shared-editor]
encounters: 3
lastSeen: 2026-09-30T00:00:00Z
rice: [2, 2, 1, 1]
priority: 33
---

# LIVE_INSTANCES_WOULD_BE_REINSTANCED refusal advises closing the map and hides what it counted

Follow-up to IN-REVIEW `E-compile-reinstances-live-instances-no-guard`. That guard works and fires live. This ticket covers only what the refusal says.

The message is at `Source/PinWright/Private/Handlers/Blueprint/BlueprintReinstancingGuard.cpp:195-198`. The `allowReinstancing` param doc repeats it (`BlueprintReinstancingGuard.h:139-145`). The message does name `allowReinstancing=true` first. The reporter's claim that it is not offered is wrong. The defect is the listed alternative, "or stop PIE / close the map first":

1. **Closing the map is the worst option in a shared editor.** The guard exists because the instances may sit in another agent's open map (`.h:38-44`). Unloading that map hurts the owner more than reinstancing its actors does. When `pieActive:false`, the tick-position hazard is already handled by safe-point family K (`.h:32-36`), so only a level dirty remains. Agents still took the map-close route: one opened `L_FootballBootstrap` just to compile `Ultra_Dynamic_Sky`.
2. **The 0-actor case is real, but the refusal is opaque about it.** The refusal fires on `InstanceCount > 0` (`.cpp:183`, `IsEmpty()` at `.h:99`). That count includes every non-CDO, non-archetype UObject of the class whose outer is an Editor, PIE or Game world (`.cpp:62-97`). Components, widget instances and other non-actor objects therefore refuse with `actorCount:0`. The text still claims "each owning level marked dirty" (`.cpp:196`). `DescribeSurvey` (`.cpp:152-155`) and the `reinstanced` block (`.cpp:117-134`) give only counts and world names, never object paths, so the caller cannot tell what the instances are.
3. `blueprint.compile` refuses before it looks at `BP->Status` (`BlueprintCompileHandler.cpp:42-52`). An agent cannot tell whether the compile it was refused would have changed anything.

**Workaround:** Re-issue with `allowReinstancing:true` when `pieActive` is false and the named worlds are your own maps (or the rebuild is acceptable). Never unload a map just to get past the guard.
**Fix:** Change the message to drop "close the map". It should recommend `allowReinstancing:true` for editor-only surveys, and stopping PIE only when `pieActive` is true. Add up to N sample instance paths and classes to each `worlds[]` entry and to `DescribeSurvey`. When `actorCount == 0`, stop claiming level dirtying. Consider reporting instead of refusing when the survey holds no actors and no components.

## History
- `#1-refusal-advice-and-opacity` `OPEN` reporter — Verified from source (no live MCP): the refusal text at `BlueprintReinstancingGuard.cpp:195-198` offers `allowReinstancing=true` and then "stop PIE / close the map first". The refusal gates on the total instance count (`.cpp:183`), not `actorCount`, and names no objects (`.cpp:152-155`). Session evidence: (a) `blueprint.compile` on `Ultra_Dynamic_Sky` was refused with "1 placed actor(s) ... L_FootballArena (Editor) ... or stop PIE / close the map first". The subagent opened `L_FootballBootstrap` to get past the guard, and the compile returned `UpToDate`. (b) A refusal carried `"actorCount":0,"pieActive":false`. The subagent retried with `allowReinstancing:true`. This is not a false positive, because the survey counts non-actor instances, but the payload cannot show which. (c) A widget-BP compile after `blueprint.graph.reconstruct_node` was refused with the same code. Case (a) is also the first recorded live fire of the guard that `E-compile-reinstances-live-instances-no-guard#3` could not trigger. Filed separately so that ticket can close on its own fix. No duplicate found: `B-blueprint-mutators-compile-ungated-tick-unsafe` (IN-REVIEW) covers guard adoption, not message content, and its workaround repeats the "close the map" advice.
