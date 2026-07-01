---
id: E-level-create-active-world-mismatch
title: "level.create reports a /Game/ path but leaves a transient /Temp/EditorAutomation/<name> world active, so the natural level.create -> level.save saves the wrong world"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [level, create, save, active-world, transient, discoverability, docs]
---

# level.create's reported path is not the active world its sibling level.save targets

After `level.create {levelPath:"/Game/Maps/LightingStudy"}` returns a success
payload naming the `/Game/` package it created, the **active editor world** is a
*transient* `/Temp/EditorAutomation/LightingStudy` package — a different object
from the `/Game/` path the call just echoed back. The natural next step a caller
reaches for is `level.save {}`, whose own description says it saves "the active
editor world's persistent level package" (`LevelHandler.cpp:219`). But the active
world is the transient one, so `level.save` reports `{saved:true}` while
**writing nothing to `/Game/`** — it saved the wrong world.

Nothing in either response surfaces the mismatch:
- `level.create`'s payload echoes the `/Game/` path (the thing it created) but
  does **not** disclose that the world it left *active* is the transient
  `/Temp/EditorAutomation/<name>` package, nor that `level.save` will therefore
  not target the created asset.
- `level.save`'s success payload echoes `packageName` (`LevelHandler.cpp:236`,
  `:241`) — which here is the `/Temp/...` name — but a caller who trusts the
  `{saved:true}` and the verb's "saves the active world" wording has no reason to
  inspect that field, and there is no warning that a `/Temp/`-mounted (transient,
  unsaveable-to-`/Game/`) package was just "saved".

The only way to discover the problem is to call `level.get_info {}` (no args),
notice the active `levelPath` is `/Temp/EditorAutomation/<name>` rather than the
`/Game/` path you created, and then recover with `level.save_as
{savePath:"/Game/Maps/<name>"}`. That is the entire detour this task burned.

## Why this is a distinct (process) angle

- `B-level-create-makes-wp-map` (IN-REVIEW) and `B-create-level-saved-true-no-umap`
  (IN-REVIEW) are about *on-disk*/WP-scaffold correctness of the create+save
  result (NewMap(true) -> WP map; McpSafeLevelSave OR-policy false `saved:true`).
  This ticket is the **discoverability** layer: even with those fixed, the
  reported-path-vs-active-world identity gap means the obvious `create -> save`
  pair targets a different world than the one named, and neither payload nor
  verb doc tells you. The judge's `B-level-export-writes-binary-not-t3d` only
  *mentions* the `/Temp/EditorAutomation/LightingStudy` active world in passing;
  it does not treat the create/save active-world mismatch as the subject.
- `E-level-create-name-path-alias` (OPEN) is param-name aliasing; orthogonal.

## What it should do

Make the active-world identity load-bearing in the response, not something the
caller must reverse-engineer:
- `level.create` should echo the **active world's** package path alongside the
  created `levelPath` (e.g. `activeWorld:"/Temp/EditorAutomation/<name>"`) — or,
  better, leave the *created* `/Game/` world active so `level.create -> level.save`
  is coherent — and/or carry a `note` that `level.save` will not persist to the
  created path until the world is saved-as / loaded there.
- `level.save` should flag when the active world is a transient `/Temp/`-mounted
  package (e.g. a `transient:true` / warning field) instead of returning a bare
  `{saved:true}` for a world that cannot land at any `/Game/` path.

## Workaround (what this task did)

`level.create {levelPath:"/Game/Maps/LightingStudy"}` -> `level.spawn_light` x3
-> `level.save {}` (reported `saved:true` but saved the transient world) ->
`level.get_info {}` revealed active world `=/Temp/EditorAutomation/LightingStudy`
-> `level.save_as {savePath:"/Game/Maps/LightingStudy"}` -> `level.get_info {}`
now reports `/Game/Maps/LightingStudy` active and `asset.exists` -> true. Until
the docs/payloads disclose the mismatch, the safe pattern is: after `level.create`
to a `/Game/` path, always `level.save_as` (not `level.save`) and verify the
active world via `level.get_info {}`.

## Docs/wiki overlay to improve

`docs/wiki-src/level.md` — the `level.create` and `level.save` sections should
state that `level.create` leaves a *transient* `/Temp/EditorAutomation/<name>`
world active (distinct from the reported `/Game/` path), that `level.save` saves
*that* active world (so it will not persist to the created `/Game/` path), and
that the correct first persistence after `level.create` is `level.save_as
{savePath:"/Game/Maps/<name>"}`, confirmed with `level.get_info {}`. (The
handler-payload changes above are the durable fix; the wiki edit is the
discoverability stopgap and is a downstream wiki process, not part of this audit.)

## History
- `#2-already-fixed` `IN-REVIEW` developer — Already resolved in current source; no code change. The ticket's load-bearing premise — that `level.create {levelPath:/Game/Maps/X}` leaves a *transient* `/Temp/EditorAutomation/<name>` world active, so `level.save {}` saves the wrong world — does not reproduce against the present tree. Verified end-to-end: (1) `level.create` calls `GEditor->NewMap(false)` (`LevelHandler.cpp:405`, now under `Handlers/Level/`), keeps it active via `SetCurrentWorld(NewWorld)` (`:407`), then `FEditorFileUtils::SaveMap(NewWorld, SavePath)` (`:416`). (2) `SaveMap` invokes `SaveWorld(..., /*bRenamePackageToFile=*/true, ...)` (engine `FileHelpers.cpp:3402-3407`). (3) With that flag `SaveWorld` RENAMES the in-memory world's package to the `/Game/` destination (`FileHelpers.cpp:1158`) and renames the world object (`:1175`), so after a successful create the active world's `World->GetOutermost()->GetName()` IS the `/Game/` path — and `level.save {}` reads exactly that name (`LevelHandler.cpp:235`), targeting the created asset. No mismatch. (4) The observed `/Temp/EditorAutomation/<name>` is NOT an engine map name: `NewMap` creates the world via `CreatePackage(nullptr)` + world name `Untitled` → a `/Temp/Untitled*` transient name (`EditorServer.cpp:2195-2198`); the `/Temp/EditorAutomation/<name>` literal appears nowhere in the engine save path nor in plugin `Source/` (all `EditorAutomation/...` hits are `Saved/EditorAutomation/...` filesystem dirs). That name was the WP **temp-package** artifact (`bIsTempPackage` branch, `FileHelpers.cpp:1105`) reachable only under the OLD `NewMap(true)` World-Partition path. (5) That path was already replaced by the landed `NewMap(true)->NewMap(false)` fix in commit `501fed6` (B-level-create-makes-wp-map), present in this tree (`git log -S "GEditor->NewMap(false)"` → 501fed6). With the non-WP world, `SaveMap`'s rename makes the active world == the `/Game/` package, eliminating the transient-active symptom this ticket was built on (the symptom report was borrowed from B-level-export-writes-binary-not-t3d's repro, which ran on the pre-fix WP build). A tester should confirm: `level.create {levelPath:/Game/Maps/X}` → `level.get_info {}` reports `activeLevelPath:/Game/Maps/X` (not `/Temp/...`) and `level.save {}` lands the `/Game/` asset.
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a "create + light + save + export" level task (story: set up /Game/Maps/LightingStudy three-point rig). Call log shows the friction directly: `level.create {levelPath:/Game/Maps/LightingStudy}` (ok) -> 3x `level.spawn_light` -> `level.save {}` returned `{saved:true}` but saved the transient world; the agent then called `level.get_info {}` and saw the active world was `/Temp/EditorAutomation/LightingStudy` (NOT the reported `/Game/` path), and recovered with `level.save_as {savePath:/Game/Maps/LightingStudy}` (`saved:true`), after which `level.get_info {}` showed `/Game/Maps/LightingStudy` active and `asset.exists` -> true. Friction note verbatim: "(1) level.create makes a transient /Temp/EditorAutomation/<name> world the active one, NOT the /Game/ package it reports creating, so level.save silently saved the wrong (Temp) world and nothing landed on disk; I only caught this by calling level.get_info with no args and seeing /Temp, then recovered with level.save_as." Code-grounded: `level.save` targets `GEditor->GetEditorWorldContext().World()` and echoes that world's `packageName` (`LevelHandler.cpp:228-241`); its description claims it saves "the active editor world" (`:219`). Distinct from `B-level-create-makes-wp-map`/`B-create-level-saved-true-no-umap` (on-disk/WP correctness of the result, IN-REVIEW), from `B-level-export-writes-binary-not-t3d` (judge-filed export-format bug, only mentions /Temp in passing), and from `E-level-create-name-path-alias` (param aliasing) — this is the reported-path-vs-active-world discoverability gap that misroutes the natural create->save sequence. Wiki overlay to improve: `docs/wiki-src/level.md`.
