---
id: B-environment-build-dispatcher-rejects-forwarded-params
title: "environment.build legacy dispatcher declares only 'action', so every cross-dispatched verb is unreachable"
status: IN-REVIEW
severity: Low
category: bug
tags: [environment, dispatcher, legacy, unreachable-surface]
encounters: 1
lastSeen: 2026-08-13T06:20:00Z
---

# environment.build legacy dispatcher rejects the params it is supposed to forward

`environment.build` (`Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:295`) is a
legacy dispatcher that cross-dispatches a named sub-action to a modern verb — e.g.
`create_procedural_terrain` → `landscape.create_procedural_terrain` (`:414-417`),
`modify_heightmap` → `landscape.edit` (`:401-404`), `set_landscape_material` →
`landscape.set_material`, `create_landscape_grass_type` → `landscape.create_grass_type`,
`generate_lods` → `asset.generate_lods`.

It declares exactly one parameter, `action`. The dispatcher's `UNKNOWN_PARAMS` validation therefore
rejects the call **before** the cross-dispatch runs, for any target verb that needs arguments — which
is all of them.

## Repro (UE 5.8, 2026-08-13)

```
call("environment.build", {action:"create_procedural_terrain",
                           landscapeName:"DotaTerrain", layerName:"Grass"})
-> [UNKNOWN_PARAMS] Unknown parameter(s) for 'environment.build':
   [landscapeName, layerName]. Valid parameters: [action].
```

`{sub:"create_procedural_terrain", ...}` fails earlier still with
`MISSING_REQUIRED_PARAM 'action'`, so `action` is the right key and the payload is still rejected.

The whole legacy compatibility surface is consequently dead: the forwarding code at `:401-425` cannot
be reached with a usable payload. `Ctx.GetRawPayload()` is passed on faithfully — the request just
never gets that far.

## Note on a related misconception

A bare `environment.create_procedural_terrain` is **not** registered and returns `UNKNOWN_ACTION`
(suggesting `environment.build.create_procedural_terrain`, `landscape.create_procedural_terrain`, …).
Some notes describe it as a live dispatch alias for the landscape paint verb; it is not, and appears
never to have been. Separately, `environment.build.create_procedural_terrain`
(`EnvironmentHandler.cpp:746`) is a genuinely different verb that spawns a procedural terrain **mesh**
actor — do not confuse the three.

## Options

1. Declare the params as pass-through / opt the dispatcher out of `UNKNOWN_PARAMS` validation so
   forwarding works as intended.
2. Retire the dispatcher and the dead forwarding block, and let `UNKNOWN_ACTION`'s existing
   did-you-mean list point callers at the modern verb (it already does).

Option 2 is probably right — the modern verbs are documented and discoverable, and a compatibility
shim that has never been callable has no users to preserve.

## History
- `#1-found-during-integration-verification` `OPEN` tester — Found while attempting the alias-parity
  check for `B-create-procedural-terrain-paints-nothing`. Pre-existing and untouched by that fix
  (`git diff` confirms no change to the dispatcher or the forwarding block); the landscape paint verb
  itself works correctly under its real name.
- `#2-option-2-chosen-docs-landed-code-deletion-staged` `IN-REVIEW` developer — **Decision: Option 2 (retire the dispatcher). Option 1 is rejected.**

  **Why Option 1 is wrong, not merely worse.** Declaring the union of all 19 arms' params would NOT make the dispatcher work, because there is a *second, independent* rejection this ticket did not record. 13 of the 19 arms forward `Ctx.GetRawPayload()` verbatim, and `CrossDispatchEnv` (`EnvironmentHandler.cpp:82-91`) routes through `UPinWrightSubsystem::DispatchMethod` → `FRpcDispatcher::DispatchMethod`, which runs the SAME `ValidateHandlerParams` against the TARGET verb's spec. The `action` key itself is in that forwarded payload and no target verb declares `action`, so the forwarded call is rejected at the far end too. Even a bare `{action:"create_sky_sphere"}` fails — `environment.build.create_sky_sphere` declares only `name` (`:130-133`). So Option 1 means declaring a 19-arm union AND stripping `action` from 13 raw-forward payloads: that is writing new, never-executed functionality, not repairing a regression. `#1`'s repro understated the problem.

  **The prior removal veto is void.** `Docs/rpc-hard-removal-rejected-candidates.md:18` had rejected removal because "not every action arm is a pure alias; some translate legacy schemas". That reasoning presumes the arms RUN. They never have, so there is no behaviour to classify or preserve. Updated that row to `REJECTION VOID; REMOVAL APPROVED, NOT YET EXECUTED` with the full argument, following the document's existing `REJECTION VOID` precedent (`physics.configure_vehicle`, whose rejection likewise assumed non-existent machinery worked).

  **Caller sweep before deciding — no real callers.** `Content/Python` (incl. `mcp_proxy.py`): zero hits. Public/customer surface: none. Only references were docs, this board, and one test. The one test, `Tests/World/TestEnvironmentHandlers.cpp:130-144` (`PinWright.environment.build.MissingRequiredParam`), calls `Reg.Func(Ctx)` directly and therefore BYPASSES `ValidateHandlerParams` — despite its name it asserts nothing about `action`, only that the registration exists and does not crash. It has no value to retarget; delete it with the verb.

  **Landed now (docs only, zero build risk):**
  - `Docs/wiki-src/environment.md` prelude — it still said *"Use `call(\"environment.build\")` for generated environment assets"*, i.e. the page's own first paragraph pointed users at the broken verb that the next section then tells them not to use. This prelude is also what the wiki ROOT index reproduces, so the bad advice was on the front page. Rewritten to route to `environment.build.create_sky_sphere` and the full `environment.control.*` names.
  - `Docs/wiki-src/environment.md` body — the existing warning said the dispatcher could still forward "sub-actions that take no arguments at all". That is too generous and is itself a false claim; replaced with the two-independent-rejections explanation and "treat it as removed".
  - `Docs/rpc-hard-removal-rejected-candidates.md:18` as above.

  **Deliberately NOT landed: the C++ deletion.** The deletion is verified self-contained — `CrossDispatchEnv` (`:82-91`) has exactly 20 call sites, ALL inside the dispatcher body (`:353`-`:470`), and `REGISTER_RPC_HANDLER` at `:293-477` is a single function body with nothing else at namespace scope inside it, so removing `:82-91` + `:293-477` + the test at `TestEnvironmentHandlers.cpp:130-144` compiles or fails as one unit. It was still not executed, because this pass was explicitly barred from building and shares the checkout with agents who ARE building; an unverifiable ~190-line deletion that leaves e.g. an unused-static warning (warnings-as-errors under UBT) would break THEIR build, and the deletion buys no user-facing improvement — the surface is already unreachable and, as of this pass, documented as unusable. Whoever next holds the build should execute exactly those three deletions and confirm the `environment` namespace page still renders with `environment.build.*` sub-verbs intact.
