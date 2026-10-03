---
id: B-list-subsystems-doc-world-param
title: "runtime-uobject-inspection wiki page documents `system.inspect.list_subsystems {world: \"pie\"}`, but the handler declares only `scope` and rejects `world` with UNKNOWN_PARAMS"
status: OPEN
severity: Low
category: bug
tags: [wiki, docs, system-inspect, list-subsystems, world, unknown-params, pie]
encounters: 2
lastSeen: 2026-09-28T10:27:00Z
rice: [1, 2, 1, 1]
priority: 17
---

# The worked example on the inspection guide does not run

`Saved/PinWright/wiki/runtime-uobject-inspection.md` (generated from `docs/wiki-src/runtime-uobject-inspection.md`)
says in Step 2 that World/GameInstance/LocalPlayer scopes "resolve the world via the same `editor|pie|auto`
machinery as Step 1" (line 34), and its worked example calls:

```
call("system.inspect.list_subsystems", { scope: "GameInstance", world: "pie" })
```

with a documented response shape `{ subsystems: [...], world: "pie", worldPath }` (lines 78-81).

The handler (`Handlers/System/SubsystemInspectHandler.cpp:31-34`) declares one parameter, `scope`, so the
dispatcher rejects the example with `UNKNOWN_PARAMS`. The generated method page
`system.inspect.list_subsystems.md` agrees with the code (only `scope`). World resolution is hard-wired to the
PIE-first default: `McpActorUtils::ResolveQueryWorld(TEXT(""), UnusedMode)` (`:104`).

Observed: UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `8fcc0b2a`, PIE running; calling it with
`world` returned `UNKNOWN_PARAMS`; dropping `world` worked because auto is PIE-first.

**Workaround:** omit `world`; auto already resolves to PIE when PIE is running.
**Fix:** either declare `world` (`editor|pie|auto`, passed to `ResolveQueryWorld`) and echo `world`/`worldPath`
as the page claims, or correct the topic page's Step 2 text, example and response shape. Declaring it is the
smaller change for callers, since the page's Step 1 promises a uniform `world` param across discovery RPCs.

## History
- `#1-world-param-rejected` `OPEN` reporter - Followed the runtime-uobject-inspection guide during a PDS PIE session; the guide's own example failed with UNKNOWN_PARAMS. Cheap (one retry without `world`).
- `#2-second-hit-wt1` `OPEN` reporter - Second sighting, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin `61c243f5`: `system.inspect.list_subsystems {scope: "GameInstance", world: "pie"}` -> `UNKNOWN_PARAMS ... Valid parameters: [scope]`, as the guide's worked example shows it. Retrying without `world` returned the PIE GameInstance subsystems. Cheap.
