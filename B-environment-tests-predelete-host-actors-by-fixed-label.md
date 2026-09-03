---
id: B-environment-tests-predelete-host-actors-by-fixed-label
title: "Environment spawn tests delete pre-existing host actors by fixed display label before creating their fixtures"
status: OPEN
severity: Critical
category: bug
tags: [tests, environment, host-safety, actor-label, wrong-target, teardown, data-loss]
encounters: 1
lastSeen: 2026-09-03T23:13:40+03:00
---

# Environment spawn tests pre-delete host actors by fixed label

## What happens

`DestroyEnvironmentSpawnTestActors()` walks the live editor world, collects every
actor of the requested class whose display label matches, and calls `Destroy()` on
all of them
(`Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp:2214-2234`).
`InvokeEnvironmentSpawnAndValidateResponse()` runs that destructive sweep before
invoking the handler (`:2255-2267`). Four callers supply fixed labels:
`McpTestSkyAtmosphere` (`:2370-2379`), `McpTestVolumetricCloud` (`:2412-2421`),
`McpTestSphereCapture` (`:2452-2463`), and `McpTestBoxCapture` (`:2495-2506`).

The `environment.build.create_sky_sphere.ResolvesEngineClass` test is a stronger
instance: it explicitly deletes every live `AActor` labelled `SkySphere` before
the call and repeats the sweep afterward (`:2546-2570`). `SkySphere` is also the
production verb's documented default label
(`Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:130-160`),
so this is a normal host-project label, not a test-owned namespace.

These tests run in `GEditor->GetEditorWorldContext().World()`. A host actor with
one of those labels is therefore destroyed before the test has created or
identified any fixture. A later spawn/destroy cycle cannot restore the original
actor or its authored properties.

## Why it matters

Running the suite can remove real atmosphere, cloud, reflection-capture, or sky
actors from the user's open map. The handler spawn also mutates the same level,
so a later map save can persist the missing host actor. The test reports only its
own spawn assertions; it never reports that unrelated pre-existing actors were
deleted.

Severity is **Critical**: the failure path directly destroys shared host-world
content, and the default `SkySphere` collision requires no unusual naming choice.

## What should happen

Never pre-delete by display label. Give every test a GUID-suffixed label and
declare `FScopedEditorWorldActorGuard` before the first handler call. Resolve the
created actor from the response's canonical `actorPath` where available; for
`create_sky_sphere`, pass its supported `name` parameter and identify the single
actor added after the guard snapshot (or add the standard actor verification
fields). Let the guard destroy only actors absent from its pre-test snapshot,
deselect them, and restore the level package's prior dirty state
(`Tests/TestWorldUtils.h:58-137`). Add the environment handler calls to the
existing guarded-spawn structural ratchet.

## Workaround

Run these tests only in a disposable empty map. Before saving any map after a
run, verify that its original environment actors still exist; restore missing
tracked map content from version control.

## Related

- Catalog `dataloss/3 wrong-target-identity-or-fallback` and `falsesuccess/7 wrong-target-scope-or-identity` — destructive selection uses a non-unique label instead of canonical identity.
- Catalog `dataloss/8 cross-caller-global-state-leak` and `crash/13 test-fixture-lifetime-is-not-scoped` — automation mutates the shared editor world without scoped ownership.
- `B-tests-spawn-live-world-no-guard` — same guard is reusable, but its current census/fix does not cover these five bodies and a guard alone cannot restore actors explicitly pre-deleted here.
- `B-tests-wipe-host-map-foliage` — sibling shared-world destructive-test hazard.

## History

- `#1-fixed-label-preclean-destroys-host-actors` `OPEN` reporter — Source-only pattern scan of the three requested test trees. Read the helper, all five callers, the environment spawn responses, `AActor::SetActorLabel`, and `FScopedEditorWorldActorGuard` end to end. Confirmed that the helper deletes pre-existing live-world actors before the handler call and that the sky-sphere case uses the production verb's ordinary default label. Deduped against the full board: `B-tests-spawn-live-world-no-guard` covers leaked new actors/stale selection and does not cover restoring host actors deliberately deleted by this helper. No build, test, editor, Saved-file access, plugin edit, existing-ticket edit, or commit was performed.
